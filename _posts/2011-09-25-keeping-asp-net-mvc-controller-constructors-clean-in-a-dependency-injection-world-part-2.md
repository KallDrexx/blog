---
layout: post
title:  "Keeping Asp.NET MVC Controller Constructors Clean In a Dependency Injection World - Part 2"
---

Previously I wrote a post an article about a way to keep Asp.net MVC constructors clean with heavy dependency injection.  In that post, which can be found [here]({{ site.baseurl }}{% post_url 2011-05-29-keeping-asp-net-mvc-controller-constructors-clean-in-a-dependency-injection-world %}), proposed creating the following class to resolve dependencies on demand:

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

This seemed like a good idea in theory, even though I got told by people on here and on Ayende's blog that it was a bad idea.  After using this in a project, I can now agree that this is a bad idea.

<h2>Dependency Ambiguity</h2>

One problem is that it doesn't make it clear what dependencies a class has, and those ambiguities are hidden behind the implementation details.  At first this doesn't seem like a big deal, but it can cause confusion down the road.  An example of this is when you have a method that retrieves data about a post in your blog system.  When you write a unit test this might fail on a private blog because it makes a call to another class to do a security check (make sure the user has access to the blog or post).  However, there is no way to know what service class is going to be used for this call without knowing the implementation details of how the IServiceFactory.GetService() call is made.  Instead, when this service class interface is passed in via IoC into the constructor, it is clear to any outside users what service interface is required to use.

<h2>IoC Testing</h2>

Another (major in my opinion) issue is you cannot easily test that ALL dependencies can be resolved.  In order to unit test that all of my dependencies can be resolved by IoC, I have to explicitly create a unit test for each type of class I *could* possibly use in my business layer or presentation layer.  I say could because since these dependencies are not being passed in through constructors I don't know exactly what IServiceFactory.GetService calls will be made.  The only way to be 100% sure I added everything to my IoC is by manually running through every path of my application.  This is error pron and just bad.

Instead, when you pass dependencies into your constructors, testing that ALL dependencies can be resolved is simple.  All you have to do is create one unit test per MVC constructor that looks like:

{% highlight csharp %}

[TestMethod]
public void Windsor_Can_Resolve_HomeController_Dependencies()
{
	// Setup
	WindsorContainer container = new WindsorContainer();
	container.Kernel.ComponentModelBuilder.AddContributor(new SingletonLifestyleEqualizer());
	container.Install(FromAssembly.Containing<HomeController>());

	// Act
	container.Kernel.Resolve(typeof(HomeController));
}
{% endhighlight %}

This will not only verify that all my HomeController dependencies can be resolved, but that all child and grandchild dependencies can be resolved as well.  This is MUCH less error prone.

<h2>So What Now</h2>

So now we come to the question that brought me here in the first place, how do I control constructor bloat when using dependency injection?  After thinking about it, it really feels like I have been trying to squash an ant by using a bulldozer.  The REAL problem with constructor bloat is that your MVC controllers are doing too much.  Once you begin to realize this, the stupidity of my ServiceFactory stuff comes clear. Constructors doing too much have more issues than just constructor bloat, it also makes it hard to navigate through controller actions.

So the real solution to the problem?  Utilize areas and spread out your constructor responsibilities among multiple MVC controllers instead of trying to fit all the actions into one controller, even if they seem vaugely related.  This will keep your MVC application much more maintainable in the long run, and easier to debug.