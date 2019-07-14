---
layout: post
title:  "A ViewModel Based Data Access Layer – Persisting Data"
---

So far in my design of a Data Access Layer that I outlined in my last [two]({{ site.baseurl }}{% post_url 2011-08-13-viewmodel-based-data-access-layer-optionally-consolidated-query-classes %}) [posts]({{ site.baseurl }}{% post_url 2011-08-07-design-of-a-viewmodel-based-data-access-pattern %}) dealt with retrieving data from the database, so now I want to explore the second half of a data access layer, how you persist data in the database.

<h2>Facade Of Simplicity</h2>

Contrary to what the repository pattern would have you believe, saving data in a data store isn't always a simple process that is the same in all situations.  Sometimes persisting data in the database requires special optimizations that are only valid in specific scenarios.  To illustrate this, let's again look at a process of saving a question at <a href="www.stackoverflow.com">Stack Overflow</a> with a traditional ORM.

When a user wants to add a question the process is simple, we just create a new instance of the question POCO and tell our ORM to save it to the database.  We want to save all of it's data because it's all new.  However what about when the use wants to edit his question?  In all ORMs that I know of you must run a query to retrieve the record from the database, populate your POCO with the data, save the changes the user made to the question, then send the modified object back to the database for saving.  

This has several inefficiencies to it.  For starters it requires you to completely retrieve the data for the question from the database when none of it is actually relevant to us.  To save the edits for a question we don't care about how many upvotes it has, how many downvotes, the original date of posting, the user who made the original post of the question, we don't even care what the original question was.  All we care about is the new title, text, and tags for the question and we want those persisted, so why retrieve all that data.  This may seem insignificant but when your application gets a lot of traffic this can cause a lot of repeated and unneeded traffic between your database and webserver.  Since the database is usually the bottleneck in a web application, and there are situations you can be in with Sql Azure where you pay for DB bandwidth, this can end up costing you in the long run.  Also consider the effect of having to mass update multiple records. 

Most ORMs (or any one worth it's salt) have some methods around this.  The first usually involves creating a new POCO object with the modified data and telling the ORM that it should save the object as is.  This is usually bad because it requires the POCO to already know all the data it probably shouldn't, such as the number of upvotes, date it was created, etc..  If any of these properties aren't set, then using this method will cause them to be nulled or zeroed out and will most likely cause data issues.  It is very risky to do this, at least with Entity Framework.  Another way around the inefficiency is to tell the ORM to use straight SQL to update the record, thus bypassing the the safety-net and security of ORM.

Both of these methods have their individual situations where they are beneficial, but rarely do you want to use one or the other all the time.  Trying to abstract each of these situations into a data access layer that is database agnostic isn't simple.

<h2>Persisting Data</h2>

So now the question becomes, how can I persist data in my ViewModel based DAL.  To do this I have come up with the following interface:

{% highlight csharp %}
public interface INonQuery<TViewModel, TReturn>
{
	TReturn Execute(TViewModel viewMode);
}
{% endhighlight %}

This interface states that you want to take the data from a specific view model and save it to the database, with a requested return type.  This allows the developer to do performance optimizations on a situation by situation basis, but if need-be they can keep the implementation consolidated until they feel they need that optimization without breaking the application layer using the API