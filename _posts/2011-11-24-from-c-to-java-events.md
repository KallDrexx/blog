---
layout: post
title:  "From C# To Java: Events"
---

I have been developing in C# for the last 3-4 years, and before that I was primarily a php coder.  During one semester of my freshman year of college I did a little Java, but not much, and I have not done any since.  I have decided to work on an N-Tier application in Java to give me a good project under my belt and show that I can use Java (and not just say to people that I can learn Java).  

The problem with going from C# to Java is that the language is very similar.  This is a problem because while I can figure out the language itself pretty easily, most "getting started" guideand tutorials are too beginner level to really be helpful.  However, jumping straight into something such as a <a href="http://www.springsource.org/tutorials">Spring framework tutorial</a> or many other more high level tutorials can be pretty overwhelming, as they all assume a general knowledge of Java concepts that are not immediately obvious by just looking at a C# to Java language guide.  Therefore, I decided to write a series containing conceptual differences or stumbles I find on my journey through implementing a Java application.

<h2>Events in C#</h2>

The first conceptual hurdle I came across was dealing with Events.  In C#, events are pretty easy to deal with, as an event is can be described as a collection of methods that all conform to a single delegate's method signature.  An example of this, <a href="http://www.codeproject.com/KB/cs/simplesteventexample.aspx">based on a good tutorial on Code Project </a> is:

{% highlight csharp %}
using System;
namespace wildert
{
    public class Metronome
    {
        public event TickHandler Tick;
        public EventArgs e = null;
        public delegate void TickHandler(Metronome m, EventArgs e);
        public void Start()
        {
            while (true)
            {
                System.Threading.Thread.Sleep(3000);
                if (Tick != null)
                {
                    Tick(this, e);
                }
            }
        }
    }
	
    class Test
    {
        static void Main()
        {
            Metronome m = new Metronome();
            m.Tick += HeardIt;
            m.Start();
        }
		
		private void HeardIt(Metronome m, EventArgs e)
		{
			System.Console.WriteLine("HEARD IT");
		}
    }
}
{% endhighlight %}

This is essentially adding the HeardIt method to the Metronome.Tick event, so whenever the event is fired, it calls theTest.HeardIt() method (along with any other method attached to the event).  This is pretty straightforward in my (biased) opinion.

<h2>Events In Java</h2>

I started reading some pages on the net about how events in Java are handled.  The first few articles I came across were kind of confusing and all over the place.  I felt like I almost understood how it's done but I was missing one piece of crucial information that binds it all together for me.  After reading a few more articles, I finally had my "Ah ha!" moment, and it all clicked.

The confusion was due to that fact that unlike in C#, there is no event keyword in Java.  In fact, and this is the piece I was missing, there essentially is no such thing as an event in Java.  In Java you are essentially faking events and event handling by using a standard set of naming conventions and fully utilizing object-oriented programming.

Using events in java is really just a matter of defining an interface that contains a method to handle your event (this interface is known as the listener interface).  You then implement this interface in the class that you want to handle the event (this is the listener class).  In the class that you wish to "fire off" the event from you maintain a collection of class instances that implements the listener interface, and provide a method so listener classes can pass an instance of themselves in to add and remove them from the collection.  Finally, the act of firing off events is merely going through the collection of listener classes, and on each listener call the listener interface's method.  It's essentially a pure-OOP solution masquerading as an event handling system.

To see this in action, let's create the equivalent of the C# event example in Java.

{% highlight java %}
package com.kalldrexx.app

// Listener interface 
public interface MetronomeEvent {
	void Tick(Date tickDate);
}

// Listener implementation
public class MainApp implements MetronomeEvent {

    /**
     * @param args the command line arguments
     */
    public static void main(String[] args) {
        EventFiringSource source = new EventFiringSource();
		source.addMetronomeEventListener(this); // Adds itself as a listener for the event
		source.Start();
    }
	
	public void Tick(Date tickDate)
	{
		// Output the tick date here
	}
}

// Event source
public class EventFiringSource {

	// Our collection of classes that are subscribed as listeners of our
    protected Vector _listeners;
	
	// Method for listener classes to register themselves
	public void addMetronomeEventListener(MetronomeEvent listener)
	{
		if (_listeners == null)
			_listeners = new Vector();
			
		_listeners.addElement(listener);
	}
	
	// "fires" the event
	protected void fireMetronomeEvent()
	{
		if (_listeners != null && _listeners.isEmpty())
		{
			Enumeration e = _listeners.elements();
			while (e.hasMoreElements())
			{
				MetronomeEvent e = (MetronomeEvent)e.nextElement();
				e.Tick(new Date());
			}
		}
	}
	
	public void Start()
	{
		fireMetronomeEvent();
	}
}
{% endhighlight %}

When the app starts (and enters the MainApp's main() method), it creates the event source and tells it that the MainApp class should be registered as a listener for the Metronome event.  When the event source class starts, it will "fire" off the event by just looking at all classes registered that implement the MetronomeEvent interface, and call the Tick() method for that implemented class.  

No special magic, just pure object-oriented programming!