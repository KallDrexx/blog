---
layout: post
title:  "Disappointed With Asp.net MVC's Testability"
---

<strong>Update:</strong> Since writing this post I found a useful tool called the <a href="http://razorgenerator.codeplex.com/">Razor Generator</a>.  Following <a href="http://blog.davidebbo.com/2011/06/unit-test-your-mvc-views-using-razor.html">this blog post</a> I am now able to effectively unit test Asp.NET MVC views.  At this point I am happy, although I don't know if this is a Razor only solution.

<br />
<hr>
<br/>

I have been happily using the Asp.NET MVC framework for personal and professional projects for about a year now.  I love how easy and straight-forward it has made web development for .NET developers, and using it has been nothing but pure joy.  However, I have noticed one nagging issue that I keep coming up against, and that issue is its advertised testability.

<h2>Background</h2>
First some background.  I have a personal website that was brought into a friends and family beta last month (unfortunately, it is not ready to be revealed to the public yet).  I was feeling good about bringing real users to the site, as it's a relatively small scale site with 198 implemented unit tests that currently test all aspects of the backend.  I also manually tested everything I could think of in the front end and everything seemed to be working great in its current state.  As a QA professional, I knew there would certainly be bugs that I missed, but I thought that they would be the less obvious bugs that came about due to real-world usage of the system.

Unfortunately, right when we brought our first user to the site I immediately got an email of a failure with the system.  I followed the user's instructions, which turned out to consist of only one step, to visit the registration page.  Upon loading the registration page I immediately saw a crash in the MVC view.  It turns out the issue was that I had moved my MVC view models from the `MyApp.Models` namespace to the `MyApp.ViewModels.Accounts` namespace for better organization.

The annoying aspect of this error was not the error itself, I merely missed the registration page when I reorganized my view models.  The annoying part was that none of my existing unit tests failed due to this.  Even if you use MVCContrib's <code>AssertViewRendered()</code> method, a compile or run-time error in a view will <strong>not </strong>cause the test to fail.

It turns out that neither MVCContrib's <code>AssertViewRendered()</code> nor any other example actually executes an action's view to make sure it actually renders. All it does is make sure it returns a <code>ViewResult</code> data type.

<h2>Lack of Testability</h2>
As far as I can tell, there is no way to test for errors in an Asp.NET MVC view. Some people are probably saying in their heads right now that I could just enable compile error checking via the <code>MVCBuildViews</code> property in the project's configuration management. There are two problems with this idea.

The first issue is that compiling views on every recompile causes the compile process to be ridiculously slow, and it's impracticable to do this all the time during development. It's possible to have a special configuration setup just for precompiling the views, but it requires the developer to remember to test-compile the project under that configuration prior to deployment. From my experiences in QA, requiring a developer to remember to do something like this is asking for trouble, and even I have forgotten to recompile my code under the special configuration prior to deployment.

The second issue with relying on <code>MVCBuildViews</code> is that it does not help find runtime errors that are in views. For example, you can test that your controller is sending a view model of type <code>MyApp.ViewModels.Accounts.RegistrationViewModel</code>, but it does nothing to make sure the view is actually expecting a view model of that type, and if the view is coded to use a different view model type then there is no way during development to know that the two are not in sync. The only way to know if the controller and view are out of sync is to manually navigate to the page in a web browser. More of these non-testable errors that can occur in a view are casting errors, bad loops, bad data in the view model, null data in the view model, etc....

In order to test for an error in a view, you have to take the returned <code>ViewResult</code> data structure and run the <code>ExecuteResult()</code> method. Unfortunately, I cannot get the <code>ExecuteResult()</code> method tested in an integration test as errors occur. For example, I have created the following test:

{% highlight csharp %}
[TestMethod]
public void RegisterResultExecutes()
{
    //arrange 
    RouteData routeData = new RouteData();
    routeData.Values.Add("action", "register");
    routeData.Values.Add("controller", "Account");

    RequestContext requestContext = new RequestContext(new MockHttpContext(), routeData);
    AccountController controller = new AccountController
    {
        FormsService = new MockFormsAuthenticationService(),
        MembershipService = new MockMembershipService(),
        Url = new UrlHelper(requestContext)
    };

    ViewResult result = controller.Register() as ViewResult;
    var sb = new StringBuilder();
    Mock<HttpResponseBase> response = new Mock<HttpResponseBase>();
    response.Setup(x => x.Write(It.IsAny<string>())).Callback<string>(y =>
    {
        sb.Append(y);
    });

    Mock<ControllerContext> controllerContext = new Mock<ControllerContext>();
    controllerContext.Setup(x => x.HttpContext.Response).Returns(response.Object);
    controllerContext.Setup(x => x.RouteData).Returns(routeData);

    //act 
    result.ExecuteResult(controllerContext.Object);
} 
{% endhighlight %}

This unit test fails with a <code>NullReferenceException</code> with the following stack trace:


{% highlight csharp %}
System.Web.Compilation.BuildManager.GetCacheKeyFromVirtualPath(VirtualPath virtualPath, Boolean&amp; keyFromVPP)
System.Web.Compilation.BuildManager.GetVPathBuildResultFromCacheInternal(VirtualPath virtualPath, Boolean ensureIsUpToDate)
System.Web.Compilation.BuildManager.GetVPathBuildResultInternal(VirtualPath virtualPath, Boolean noBuild, Boolean allowCrossApp, Boolean allowBuildInPrecompile, Boolean throwIfNotFound, Boolean ensureIsUpToDate)
System.Web.Compilation.BuildManager.GetVPathBuildResultWithNoAssert(HttpContext context, VirtualPath virtualPath, Boolean noBuild, Boolean allowCrossApp, Boolean allowBuildInPrecompile, Boolean throwIfNotFound, Boolean ensureIsUpToDate)
System.Web.Compilation.BuildManager.GetVirtualPathObjectFactory(VirtualPath virtualPath, HttpContext context, Boolean allowCrossApp, Boolean throwIfNotFound)
System.Web.Compilation.BuildManager.GetObjectFactory(String virtualPath, Boolean throwIfNotFound)
System.Web.Mvc.BuildManagerWrapper.System.Web.Mvc.IBuildManager.FileExists(String virtualPath)
System.Web.Mvc.BuildManagerViewEngine.FileExists(ControllerContext controllerContext, String virtualPath)
System.Web.Mvc.VirtualPathProviderViewEngine.GetPathFromGeneralName(ControllerContext controllerContext, List`1 locations, String name, String controllerName, String areaName, String cacheKey, String[]&amp; searchedLocations)
System.Web.Mvc.VirtualPathProviderViewEngine.GetPath(ControllerContext controllerContext, String[] locations, String[] areaLocations, String locationsPropertyName, String name, String controllerName, String cacheKeyPrefix, Boolean useCache, String[]&amp; searchedLocations)
System.Web.Mvc.VirtualPathProviderViewEngine.FindView(ControllerContext controllerContext, String viewName, String masterName, Boolean useCache)
System.Web.Mvc.ViewEngineCollection.&lt;&gt;c__DisplayClassc.b__b(IViewEngine e)
System.Web.Mvc.ViewEngineCollection.Find(Func`2 lookup, Boolean trackSearchedPaths)
System.Web.Mvc.ViewEngineCollection.FindView(ControllerContext controllerContext, String viewName, String masterName)
System.Web.Mvc.ViewResult.FindView(ControllerContext context)
System.Web.Mvc.ViewResultBase.ExecuteResult(ControllerContext context)
MyApp.Tests.Controllers.AccountControllerTest.RegisterResultExecutes() in C:\Users\KallDrexx\Documents\Projects\MyApp\MyApp.Tests\Controllers\AccountControllerTests.cs: line 303
{% endhighlight %}

I can't tell if it's possible to get any further from here, and I am beginning to think that creating an integration test on views is impossible. This makes me a bit sad and confused, sad because Asp.NET MVC heavily advertises the fact that it's extremely testable and confused because I can't seem to find any discussion on this at all, and I can't believe that I am the only one interested in having automated tests that verify exceptions do not occur when rendering a view. If this isn't testable then I am not quite sure how MVC is any more testable than a well architected webforms application.

As of now I will probably have to either have unit tests that use <code>HttpWebRequests</code> (if it's possible due to parts of the site requiring forms authentication) or tests that use the Watin framework, neither of which are particularly appealing and add additional points of possible failure.