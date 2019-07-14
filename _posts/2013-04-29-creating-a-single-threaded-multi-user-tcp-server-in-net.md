---
layout: post
title:  "Creating a Single Threaded Multi-User TCP Server In .Net"
---

In my random desire to build a simple IRC server I decided the first step was to build a TCP server that can read data and send data simultaneously to multiple users.  When someone first starts to write a networked server they quickly realize it's not as easy as they had hoped due to all the network operations being done via blocking method calls.  Before .Net 4.5, handling multiple socket connections meant having to put the socket handling calls into alternate threads, either by AsyncCallback calls or manually loading the method calls into the thread pool.  These then require thread syncing when data needs to get back to the main thread so that the server's business logic can run on the socket events.

With the introduction of the Async/Await keywords in .Net 4.5, it has become much easier to run your socket operations asynchronously while never leaving the main application thread.  If you are not familiar with the Async/Await keywords it would probably be a good thing to read <a href="http://blog.stephencleary.com/2012/02/async-and-await.html" title="Stephen Cleary's tutorial">Stephen Cleary's tutorial</a>.

So here is how I went about making a single threaded TCP server that can handle multiple clients.  This may not be the best way, but it did seem to be the most successful attempt I had for this.

<h2>Network Clients</h2>

The first thing I needed was a class to handle already connected clients.  I needed the class to be able to take an already established connection and have it be able to receive network messages (and pass them to the business logic for processing) as well as sending messages back to the client.  It also needs to determine when a connection has been dropped (either gracefully or not).  Most importantly, these actions should not block the general server flow.  As long as the server's business logic processor is not actually processing a network message, it should not be waiting on a networked client.

Since both receiving messages from the client and a client being disconnected are important aspects that the underlying server should be aware of, I defined the following delegates:

{% highlight csharp %}
public delegate void MessageReceivedDelegate(NetworkClient client, string message);

public delegate void ClientDisconnectedDelegate(NetworkClient client);
{% endhighlight %}

I then created the skeleton of the NetworkClient like so:

{% highlight csharp %}
public class NetworkClient
{
    private readonly TcpClient _socket;
    private NetworkStream _networkStream;
    private readonly int _id;

    public bool IsActive { get; set; }
    public int Id { get { return _id; } }
    public TcpClient Socket { get { return _socket; } }

    public event MessageReceivedDelegate MessageReceived;
    public event ClientDisconnectedDelegate ClientDisconnected;

    public NetworkClient(TcpClient socket, int id)
    {
        _socket = socket;
        _id = id;
    }

    private void MarkAsDisconnected()
    {
        IsActive = false;
        if (ClientDisconnected != null)
            ClientDisconnected(this);
    }
    }
{% endhighlight %}

This handles creating the client (not the connection, but the class itself) and a helper method for handling when a client has been disconnected.  Now we need to listen for incoming data on for this TCP client.  Since reading TCP input is blocking, we need to perform this asynchronously with the following code:

{% highlight csharp %}
public async Task ReceiveInput()
{
    IsActive = true;
    _networkStream = _socket.GetStream();

    using (var reader = new StreamReader(_networkStream))
    {
        while (IsActive)
        {
            try
            {
                var content = await reader.ReadLineAsync();

                // If content is null, that means the connection has been gracefully disconnected
                if (content == null)
                {
                    MarkAsDisconnected();
                    return;
                }

                if (MessageReceived != null)
                    MessageReceived(this, content);
            }

            // If the tcp connection is ungracefully disconnected, it will throw an exception
            catch (IOException)
            {
                MarkAsDisconnected();
                return;
            }
        }
    }
}
{% endhighlight %}

This method essentially loops until the network client is not activate anymore and awaits for a full line of data to be returned by the client.  If a null response is returned or an IOException occurs, then that means the connection has been disconnected and I need to mark the client as such.  By making sure we are awaiting for incoming data, it assures we do not block the application flow while waiting for data to come down the pipe.  This method returns a Task, so that we can check for unhandled exceptions in the main server.

Next we need to be able to asynchronously send data to the user.  I do this with the following method:

{% highlight csharp %}
public async Task SendLine(string line)
{
    if (!IsActive)
        return;

    try
    {
        // Don't use a using statement as we do not want the stream closed
        //    after the write is completed
        var writer = new StreamWriter(_networkStream);
        await writer.WriteLineAsync(line);
        writer.Flush();
    }
    catch (IOException)
    {
        // socket closed
        MarkAsDisconnected();
    }
}
{% endhighlight %}

I do this asynchronously so we don't risk blocking the entire server while waiting for the whole TCP process to finish.  

<h2>Server Infrastructure</h2>

Now that we have a fully working class to handle networked clients, we need to create the server infrastructure.  We need some way to:
<ul>
<li>Accept new clients on a specific IP address and port</li>
<li>Turn those clients into NetworkClient instances</li>
<li>Process commands coming from network clients</li>
<li>Handle client disconnections</li>
</ul>

This begins with the following class skeleton:

{% highlight csharp %}
public class Server
{
    private readonly TcpListener _listener;
    private readonly List&lt;NetworkClient&gt; _networkClients;
    private readonly List&lt;KeyValuePair&lt;Task, NetworkClient&gt;&gt; _networkClientReceiveInputTasks; 
    private Task _clientListenTask;

    public bool IsRunning { get; private set; }

    public Exception ClientListenTaskException
    {
        get { return _clientListenTask.Exception; }
    }

    public Server(IPAddress ip, int port)
    {
        _listener = new TcpListener(ip, port); 
        _networkClients = new List&lt;NetworkClient&gt;();
        _networkClientReceiveInputTasks = new List&lt;KeyValuePair&lt;Task, NetworkClient&gt;&gt;();
    }
}
{% endhighlight %}

The _networClientRecieveInputTasks would be used to check for exceptions while listening for input from a client.  The client listen task will be used to reference the asynchronous task that listens for new client connections, and this would be use this to check for unhandled exceptions being thrown.  Everything else is to get data ready for actually running the server.

We need to consider what to do when a command is received by the client.  To get up and running quickly we are just going to relay the incoming data out to all other clients, via the following method:

{% highlight csharp %}
private async void ProcessClientCommand(NetworkClient client, string command)
{
    Console.WriteLine("Client {0} wrote: {1}", client.Id, command);

    foreach (var netClient in _networkClients)
        if (netClient.IsActive)
            await netClient.SendLine(command);
}
{% endhighlight %}

Now we need to handle the NetworkClient.ClientDisconnected event.  In this case all we want to do is close the network socket and remove the client from our list.  

{% highlight csharp %}
private void ClientDisconnected(NetworkClient client)
{
    client.IsActive = false;
    client.Socket.Close();

    if (_networkClients.Contains(client))
        _networkClients.Remove(client);

    Console.WriteLine("Client {0} disconnected", client.Id);
}
{% endhighlight %}

The next thing we need is to figure out what we want to do when a client is connected.  When a client connects we need to create a NetworkClient instance for them, assign them an identification number (for internal use only), hook into the NetworkClient's events, and start listening for input from that client.  This can be accomplished with the following method:

{% highlight csharp %}
private void ClientConnected(TcpClient client, int clientNumber)
{
    var netClient = new NetworkClient(client, clientNumber);
    netClient.MessageReceived += ProcessClientCommand;
    netClient.ClientDisconnected += ClientDisconnected;

    // Save the Resulting task from ReceiveInput as a Task so
    //   we can check for any unhandled exceptions that may have occured
    _networkClientReceiveInputTasks.Add(new KeyValuePair&lt;Task, NetworkClient&gt;(netClient.ReceiveInput(),
                                                                                netClient));

    _networkClients.Add(netClient);
    Console.WriteLine("Client {0} Connected", clientNumber);
}
{% endhighlight %}

This will take a TcpClient, create a new NetworkClient for it, tie up the events and start receiving input.

We now have everything e need to execute information from a client, we just need to actually accept incoming client connections.  This of course needs to be done asynchronously so we do not block the server flow while waiting for a new connection.

{% highlight csharp %}
private async Task ListenForClients()
{
    var numClients = 0;
    while (IsRunning)
    {
        var tcpClient = await _listener.AcceptTcpClientAsync();
        ClientConnected(tcpClient, numClients);
        numClients++;
    }

    _listener.Stop();
}

public void Run()
{
    _listener.Start();
    IsRunning = true;

    _clientListenTask = ListenForClients();
}
{% endhighlight %}

That's pretty much all the code that is needed.  Now all you have to do is add the server calls to your main method:

{% highlight csharp %}
static void Main(string[] args)
{
    var server = new Server(IPAddress.Any, 9001);
    server.Run();

    while (server.IsRunning)
    {
        Thread.Sleep(100);
    }
}
{% endhighlight %}

We need the while() loop due to the main functionality of the server running asynchronously.  Otherwise the program would immediately exit.  Now if you run the server you will be able to connect multiple telnet sessions to each other and pass messages back and forth.

<h2>Threading</h2>

The problem with our server so far is that it is <i><b>not</b></i> operating in a single thread.  This means that once you have a lot of clients connecting, sending messages, etc.. you will come up with syncing issues (especially once you start adding real business logic to the mix).  

Outside of WPF and Winforms applications async/await operate in the threadpool not in the main thread.  This means you cannot 100% predict which await operations will work on which threads (you can read more about this at <a href="http://blogs.msdn.com/b/pfxteam/archive/2012/01/20/10259049.aspx">this MSDN blog</a>).   

If you want proof of this on your sample server, you can add the following code everywhere you have any other Console.WriteLine() call:

{% highlight csharp %}
Console.WriteLine("Thread Id: {0}", Thread.CurrentThread.ManagedThreadId);
{% endhighlight %}

If you also add this to your Main() method and run the application you will notice multiple thread ids being displayed in the console.

Async and await commands utilize <a href="http://www.gamlor.info/wordpress/2010/10/c-5-0-async-feature-be-aware-of-the-synchronization-context/">the SynchronizationContext</a> which controls how (and on what threads) the different actions are run on.  Based on that reference, I created the following SynchronizationContext implementation.

{% highlight csharp %}
public class SingleThreadSynchronizationContext  : SynchronizationContext
{
    private readonly Queue&lt;Action&gt; _messagesToProcess = new Queue&lt;Action&gt;();
    private readonly object _syncHandle = new object();
    private bool _isRunning = true;

    public override void Send(SendOrPostCallback codeToRun, object state)
    {
        throw new NotImplementedException();
    }

    public override void Post(SendOrPostCallback codeToRun, object state)
    {
        lock (_syncHandle)
        {
            _messagesToProcess.Enqueue(() =&gt; codeToRun(state));
            SignalContinue();
        }
    }

    public void RunMessagePump()
    {
        while (CanContinue())
        {
            Action nextToRun = GrabItem();
            nextToRun();
        }
    }

    private Action GrabItem()
    {
        lock (_syncHandle)
        {
            while (CanContinue() && _messagesToProcess.Count == 0)
            {
                Monitor.Wait(_syncHandle);
            }
            return _messagesToProcess.Dequeue();
        }
    }

    private bool CanContinue()
    {
        lock (_syncHandle)
        {
            return _isRunning;
        }
    }

    public void Cancel()
    {
        lock (_syncHandle)
        {
            _isRunning = false;
            SignalContinue();
        }
    }

    private void SignalContinue()
    {
        Monitor.Pulse(_syncHandle);
    }
}
{% endhighlight %}

I then had to update my program's Main method to utilize the context.

{% highlight csharp %}
static void Main(string[] args)
{
    var ctx = new SingleThreadSynchronizationContext();
    SynchronizationContext.SetSynchronizationContext(ctx);

    Console.WriteLine("Main Thread: {0}", Thread.CurrentThread.ManagedThreadId);
    var server = new Server(IPAddress.Any, 9001);
    server.Run();

    ctx.RunMessagePump();
}
{% endhighlight %}

Now if you run the application and send some commands to the server you will see everything running asynchronously on a single thread.

<h2>Conclusion</h2>

This may not be the best approach, and a single-threaded TCP server is probably not the most efficient in a production environment, but it does give me a good baseline to work with to expand out its capabilities.