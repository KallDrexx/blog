---
layout: post
title:  "Scenario Based Testing"
---

As mentioned [in my previous post]({{ site.baseurl }}{% post_url 2016-02-04-the-frustrations-of-unit-testing %}) I am trying to experiment with more resilient ways to perform test driven development.  In my ideal world, not changing functionality means tests do not need to be changed.  

<h1>Recap On Unit Testing Shortcomings</h1>
Unit testing too often breaks with refactors unless the refactoring only takes place inside a single class.  More often than not you start writing code to do something in one class and then later realize you need the same logic in another class.  The proper way to handle this is to extract the logic into a common dependency (whether it is a new dependency or existing one), but if your tests are scoped to a single class (with all dependencies mocked) this becomes a breaking change requiring all of your mocks to be touched up (or new ones added) in order for your test to confirm that nothing has changed.

The problem with that is you are forcing the change in your tests to verify that functionality hasn't changed, but if you are changing your test you can't be 100% sure functionality hasn't changed.  The test is no longer exactly the same.  All you can be 100% sure about is that the updated test works with the updated code, and then you pray that no regressions come out of it.  

Expanding the scope of what you consider a unit for test purposes helps alleviate this, but that only works well if you have a complete understanding of both the code and business domain models, so that each unit becomes a domain.  This is hard to do until a product is out and in active use because it is rare the business or engineering sides have a full grasp of the product before it is in customers hands.  The product always changes after the MVP.

<h1>So What's a Solution?</h1>
Scenario based testing is a test pattern I am playing with that tests the end user boundaries of a system.  At the end of the day what I care about as a developer is that end users (whether they are people or applications) can perform actions against the system and the correct processes are run.  

This means that if I am using a web based API and my system allows users to edit a blog post, my tests will verify that sending the correct data to my API and then retrieving that post back will return the post with the correct fields edited in the correct way.  I don't care about testing that it's stored in the database in a specific way, or a specific class is used to process the logic on the post, I only care about the end result.  If the results of an action never leave the system, does it really matter if it occurs?

<h1>So It's An Integration Test?</h1>
Some of you are probably shrugging your shoulders thinking I'm talking about an integration tests.  While these can somewhat be considered as integration tests, I try not to call them integration tests as all external components are mocked out.  

Most integration tests involve writing to actual external data stores (like SQL) in order to verify that it is interacting with those systems properly.  This is still necessary, even with scenario testing, because you can never be sure that your SQL queries are correct, that your redis locks are done properly, etc...

The purpose of this style of testing though is to codify the expected functionality of your system with as much area coverage as possible, lowering the needed scope of the required integration testing to just those external components.

Mocking all external systems also has the side benefit of being able to run tests extremely quickly and in parallel.

<h1>What Do These Tests Look Like?</h1>
Another goal for me in my testing is that it's easy to read and understand what is going on, so if a developer breaks a test he has an understanding of the scope of functionality that is no longer working as expected.  

If we take the previous example of editing a blog post, a test may look like this:

{% highlight csharp %}

[Fact]
public void Can_Edit_Blog_Post()
{
    var date = new DateTimeOffset(2016, 1, 2, 3, 4, 5, TimeSpan.Zero);
    var date2 = new DateTimeOffset(2016, 1, 3, 0, 0, 0, TimeSpan.Zero)

    var originalPost = new PostDetails
    {
        Title = "My New Title",
        Content = "My Awesome Post"
    };

    var updatedPost = new PostDetails
    {
        Title = "Edited Title",
        Content = "Edited Content"
    }

    TestContext.SetDate(date)
        .AsAuthoringUser()
        .SubmitNewPostApiRequest(originalPost)
        .ExecuteAction(context => updatedPost.Id = context.GetLatestPostId())
        .SetDate(date2)
        .SubmitEditPostApiRequest(postId, updatedPost);

    var post = TestContext.GetPost(updatedPost.Id);
    post.Should().NotBeNull();
    post.Title.Should().Be(updatedPost.Title);
    post.Content.Should().Be(updatedPost.Content);
    post.LastEditedDate.Should().Be(date2);
}

{% endhighlight %}

In this you can clearly see the functional intent of the test.  As a developer you are able to completely change around the organization of the code without having any impact on the passing or failing of this test, as long as it is functionally the same as it was before your refactor.  

If this looks like magic, that's because it does require some up front development costs.  The TestContext class mentioned above is a class that holds details about the current test, including IoC functionality and other information it needs to track to perform tests.

In my current code base utilizing this pattern, the TestContext class is initialized in the test class itself, and has all the logic for how to set up IoC, what it needs to mock, and what events in the system it might need to tie into in order to be able to perform verifications.

All the methods shown on the TestContext object are merely extension methods that utilize the test context to make it easy to perform common actions.  So for example, SetDate(date) call would get the mocked date provider implementation from IoC and force it to a specific date (to allow freezing time).  

`SubmitNewPostApiRequest()` could be something that either calls on your API using a fake browser or calls your API controller, depending on what framework you are using.  The current code base I use this for uses <a href="http://nancyfx.org/">Nancy</a> for it's API layer, which provides tools that allow sending actual POST or GET requests to the API for testing purposes.

<h1>What Are The Cons?</h1>

This approach isn't perfect however.  Since we are only testing the system boundaries it requires the developer to have a decent grasp of the underlying architecture to diagnose test failures.  If a test starts failing it may not be immediately clear why a test is failing, and has the possibility to waste time while trying to figure that out.

This does force the developer to write smaller pieces of code in between running tests, so they can keep the logical scope of the changes down in case of unexpected test failures.  This also requires tests to be fast so developers are not discouraged from running tests often.

All in all, I am enjoying writing tests in this style (so far at least).  It has let me do some pretty complicated refactors as the application's scope broadens without me worrying about breaking existing functionality.