---
layout: post
title:  "Keeping Asp.NET MVC Controller Constructors Clean In a Dependency Injection World"
---

<strong>Update:</strong> After playing around with the concepts I talked about in this post I now pretty much disagree with what I wrote here.  To see why, see [part 2]({{ site.baseurl }}{% post_url 2011-09-25-keeping-asp-net-mvc-controller-constructors-clean-in-a-dependency-injection-world-part-2 %}).

<br />
<hr>
<br />

In my experience, dependency injection has proven to be invaluable when creating applications.  It has allowed me to easily follow test driven development practices, be confident that my application functions as expected, and it gives me confidence that any bugs I had previously found do not crop up again.

However, one trend that I have noticed is that as my Asp.NET MVC application becomes more complex, my controller constructors are becoming bloated with all of the service objects that the controller might use.  I say might because not all the service objects are being used in all actions, but they must be passed into the controller's constructor to facilitate unit testing.  This issue of constructor bloat is made worse due to my business/service layer being made up of many small query and command classes.  Here is an example of one of my controllers:

{% highlight csharp %}

public class ContactController : MyAppBaseController
{
    public ContactController( ContactsByCompanyQuery contactsByCompanyQuery,
                                        ContactByIdQuery contactByIdQuery,
                                        CreateContactCommand createContactCommand,
                                        EditContactCommand editContactCommand,
                                        ISearchProvider searchProvider,
                                        IEmailProvider emailProvider)
    { 
        // copy parameters into variables
    }
}

{% endhighlight %}

As you can see from this example, as I add more actions to my controller and need more service classes I have to add more parameters into my controller, which also means I have to update all unit tests to use this new parameter when instantiating the controller.  This also strikes me as inefficient as it requires the inversion of control system to always instantiate all service classes even when the executing MVC action may only need one or two.  

I needed a system that would allow me to minimize the number of parameters in my controller constructors while still giving me full flexibility with dependency injection and retaining the ability to unit test my controllers and mock my service classes.  After thinking about this problem for a bit I came up with the idea of creating a factory class that instantiates service classes on demand directly from the IoC system.  As I use Castle Windsor for IoC I created the following factory class:

{% highlight csharp %}
public class WindsorServiceFactory : IServiceFactory
{
    protected IWindsorContainer _container;

    public WindsorServiceFactory(IWindsorContainer windsorContainer)
    {
        _container = windsorContainer;
    }

    public ServiceType GetService<ServiceType>() where ServiceType : class
    {
        // Use windsor to resolve the service class.  If the dependency can't be resolved throw an exception
        try { return _container.Resolve<ServiceType>(); }
        catch (ComponentNotFoundException) { throw new ServiceNotFoundException(typeof(ServiceType)); }
    }
}
{% endhighlight %}

It turned out to be a much simpler solution than I thought it would be.  Now my constructors look like

{% highlight csharp %}
public class ContactController : MyAppBaseController
{
    protected IServiceFactory _serviceFactory;

    public ContactController (IServiceFactory factory)
    {
        _serviceFactory = factory;
    }

    public ActionResult Search(string query)
    {
        var results = _serviceFactory.GetService<ISearchProvider>().Search(query);
        return View(results);
    }
}
{% endhighlight %}

This is much cleaner in my opinion and it also allows me to completely ignore my constructors even as I add more functionality into my controllers.

I am also able to easily use this in my unit tests with mocks.  Here is an example using Moq:

{% highlight csharp %}
[TestMethod]
public void Can_Search_Contacts()
{
    // Setup
    var factory = new Mock<IServiceFactory>();
    var searchProvider = new Mock<ISearchProvider>();
    factory.Setup( x => x.GetService<ISearchProvider>()).Returns(searchProvider.Object);
    var controller = new ContactController(factory.Object);
    
    // Act
    var result = controller.Search();

    // Add verifications here
}

{% endhighlight %}