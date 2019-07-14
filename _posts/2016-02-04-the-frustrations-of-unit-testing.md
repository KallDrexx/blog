---
layout: post
title:  "The Frustrations of Unit Testing"
---

Test driven development has proven itself to be a good thing for me over the years.  It has helped me to not only codify my design decisions, but it has also helped keep me focused on the next iterative steps I need to create the larger change in the system that I originally set out to do.  It has also helped keep regressions from occurring.

Unit testing is one integral part of test driven development, as it allows you to take a unit of code and make sure it does what it is supposed to.  

<h1>What Is a Unit?</h1>
What qualifies as a unit of code differs from person to person.  Most "conventional wisdom" of the internet will lead you to believe that in object oriented programming languages, a unit is a single class.  However there is no reason that this should be the case, as you can have functional units in your code base that involve multiple classes or even a single function, depending on the full scope of what is being performed (and how logic is related to each other).  The reality of the situation though is that as you start involving more classes in your functional unit, your tests start taking on a profile that looks more like integration tests rather than unit tests.  This complicates the effort required to setup a test and seems to make me gravitate back to smaller units, which usually ends up back as a class as a single testable unit.  

Unit testing at the class level is also considered good as it makes it much quicker to figure out where the code is that is causing a test to break, as the area of code that can be causing a bug is isolated to a single class (which should be relatively small anyway).

However, as time goes on I have started disliking testing with classes as the boundary of the test.

<h1>An Example</h1>

Let's say we are writing code that contains the business logic for updating a post on a blog.  A minimal version would contain several business processes, such as:

<ul>
<li>Verify the logged in user has access to the post</li>
<li>Get the current version of the post (with categories that have already been added to the post)</li>
<li>Get the list of categories known by the system</li>
<li>Modify the post with the new details </li>
<li>Save the post to the data store</li>
</ul>

Assume each bullet point is it's own class or function that already has tests for them.  I now need to test the class that is combining each of these components into a cohesive pipeline that performs the correct operations under the correct conditions in the correct order.

So let's say I want to create a test that a user with access to a post can modify the title of a post.  At the outset it sounds like a simple case of inputs and outputs.

Unfortunately, it gets a lot more complicated than that.  Since i just want to focus on the functionality of my <code>EditPost()</code> function (since all the other functionality should already be tested and I just need to compose them together) I need to create mocks so that I don't have to test the implementation details of checking if a user has access to a post, can retrieve a post object successfully from the data store, etc...

So for this test I create my first mock that says that when I pass my user id and post id to the authorization function, it should always return true.  

The next mock I need to create says that when I pass a post id to the post retrieval function, it returns to me a post object with some pre-defined information.  

Another mock I need to create says that when I call on the function to retrieve all blogging categories, it returns to me a static list of categories.

Yet another mock I need to create says that when I call the function to save the post to the data store, it siphons off the parameters I passed into it for verification.

Now that I have those mocks setup, I can now pass them into my edit post function call (via dependency injection), call my edit post function with the new title, and verify the parameters I siphoned off are what I expect them to be.  If I am following the TDD methodology then I now have failing tests, so that I can fill in the function's implementation and get a nice passing test.

You continue on creating more tests around this function (and all the mocks that come with it) to test authorization failures, post doesn't exist conditions, other specific editing conditions (e.g. the creation date must always remain the old value on published posts, edit date is automatically set to the current date/time), etc....

<h1>So What's the Problem?</h1>

So a few days later you realize that always pulling down a post with it's full content and categories is too much.  Sometimes you only need titles, sometimes titles and categories, sometimes titles and content without categories.  So you split them out into different functions that return different pieces of the blog post (or maybe you have different functions for blogPostWithContent, blogPostWithCategories, etc...).  

Or maybe you realize you need to add some more parameters to the authorization method due to changing requirements of what information is needed to know if someone is authorized to edit a post in some conditions that may or may not be related to the tests at hand.

Now you have to change every test of your edit post function to use new mock(s) instead of the previous mocks.  However, when I wrote the original test all I really wanted to test was that an authorized user could change the title of a post but all my changes are all related to the implementation details of how that is accomplished.  

The tests are now telling me that I have a code smell, and that my tests on the logical flow of the process are too intertwined with the tests surrounding how I'm composing all my functional units.  The correct way to fix this problem is to move all the the business logic of how to modify a post object into it's own pure function.  This function would take in a data store post object, the change operations to perform on it, and the categories listing.  It would then output the resulting post that needs to be saved to the data store.

We can then have tests on the logic of editing a post on the new function and keep the tests on the original function focused solely on how all the functions are composed together. Of course we have to completely modify the previous tests now to reflect this refactoring effort, and since the old tests are no longer valid we have to be careful to create regressions since we are completely changing old tests and adding completely new tests.  

The question is, was this refactor worth it?  At an academic level it was because it gave us more pure functions that can be tested independently of each other.  Each functional unit is very small and does only one thing in a well tested way.  

However, at a practical level it thoroughly expands the code base by adding new data structures, classes, and layers in the application.  These additional pieces of code need to be maintained and navigated through when bugs or changing requirements come down the pipeline.  Additional unit tests have to then be created to verify the composition of these functions are done in the correct and expected manner.

Another practical issue that brings this to a wall is that you are usually working with a team that has various skill and code quality concern levels, combined with the need to get a product and/or feature out to customers so it can generate revenue and be iterated on further.

It also increases the need for more thorough integration tests.  An example of this is if the function that actually performs the edit requires the post to have a non-null creation date, or only is allowed to edit non-published posts.  This function will have unit tests surrounding it to make sure it surfaces the correct error when it is passed data that it considers bad, but there is no way to automatically know that this new requirement is not satisfied by callers of the function with unit tests alone, and this error will only be caught when the system is running as a holistic unit.

So what's the solution?  There really is no good solution.  Integration tests are just as important as unit tests in my opinion, but having a comprehensive set of unit and integration tests that easily survive refactoring is extremely time consuming and may not be practical to the business objectives at hand.  

Integration tests by themselves can usually withstand implementation level refactorings, but they are usually much slower to perform, in some cases are completely unable to be run in parallel, require more infrastructure and planning to setup properly (both on a code and IT level), and they make it harder to pinpoint exactly where a bug is occurring when a test suddenly fails.  
