---
layout: post
title:  "Design of a ViewModel Based Data Access Pattern"
---

I have been doing a lot of reading about data access pattern, and I haven't really been happy with them.  Since then I have been doing a lot of thinking about a good way to do data access.

<h2>The Repository Pattern</h2>
The <a href="http://www.google.com/search?q=repository+pattern">repository pattern</a> seems to be the most popular data access pattern.  The problem I have with this pattern is that it is very rigid for querying.  Generic repositories are pointless in my opinion as the business logic has to have intimate logic of the data layer in order to make efficient use of it, and if you are tying the two together you might as well deal with the data access manually in your business layer.  

Non-generic repositories get tricky as you get into more complex querying scenarios.  Most business processes your application will go through requires different information, even if the base entity it needs is the same.  If you think about <a href="www.stackoverflow.com">StackOverflow</a> as an example, on one screen you need to query for the user's information and all related questions but if you click on the "responses" link now you need the user's information with all responses the user has made.  The only way to do this without relying on lazy loading (which you don't want as lazy loading can too easily become a performance nightmare) is to create your repository with one method per query.  You will then end up with something like:

{% highlight csharp %}
public interface IUserRepository
{
	User GetUserById(int id);
	User GetUserWithResponses(int id);
	User GetUserWithQuestions(int id);
	
	// etc...
}
{% endhighlight %}

Now say you need to implement a query that retrieves the user's latest activity, which combines information from their questions <strong>and</strong> responses.  You can't user the previous two methods and you will have to implement yet another.  This can very easily get out of hand as you require more and more types of queries.  It also makes it confusing to determine which repository a query should be in (for example, if you want all the questions for a user is that in the User repository (user with associated questions) or in the questions repository (questions by user))?

<h2>Query Objects</h2>

Another pattern for querying is using Query Objects.  I first learned about these from the 2nd half of <a href="http://ayende.com/blog/3955/repository-is-the-new-singleton">this post by Ayende</a>.  The essence of this is that each class is for a single query, and to get the information you need you just call the Execute() method of the class required.  While you now have a lot classes rather than a lot of methods, I find classes easier to organize and look for than repository methods.  

The problem is that you still have to keep track of how you name your query classes, and remember exactly what data is returned by each query class.  How do you distinguish between a query class that returns a user with his questions versus a query class that returns a user with his activity.  It's hard to do without keeping a very strict naming convention, and even then it can get a bit confusing.

<h2>Querying By ViewModel</h2>

Let's stop thinking about the code for a second and think of the actual business/application logic of what we are trying to do.  All we (or at least I) really want is to run a query to retrieve specific data.  When working with my UI or business logic I don't care about the specifics of how that happens, I just want the database to give me the information I need.

My idea is a slight extension of query objects.  While each class is for one specific query, that class is not only for retrieving data from the database, but for placing that data in a specific non-persisted data class (aka a view model).  The returned class doesn't have to resemble the database's data model at all, and thus this would give us total flexibility to retrieve data exactly as how our application wants it rather than relying on how the database stores it.

I would accomplish this via the following interface:

{% highlight csharp %}
public interface IQuery<TViewModel>
{
	TViewModel Execute();
}
{% endhighlight %}

Now, let's say I want to retrieve the user's info with his questions.  My UI might use the following view model

{% highlight csharp %}
public class UserQuestionsPageViewModel
{
	public int UserId { get; set; }
	public string Name { get; set; }
	public IList<QuestionSummaryViewModel> Questions { get; set; }
}
{% endhighlight %}

The idea is that my Asp.Net MVC's controller code to run this query would look like: 

{% highlight csharp %}
public class UserController : Controller
{
	protected IQuery<UserQuestionsPageViewModel> _query;
	
	public Usercontroller(IQuery<UserQuestionsPageViewModel> query)
	{
		_query = query;
	}

	public ActionResult UserQuestions(int userId)
	{
		var model = _query.Execute();
		return View(model);
	}
}
{% endhighlight %}

The interesting aspect of this is that we don't care about how we named the query object, that part is already covered by dependency injection.  All we have to specify is that we have a specific view model and we want to fill it with data from our data store, and the query commences.  This makes our data access very simple and intuitive to use, and minimizes how many layers it takes to get data from the database into the view model for the UI.

<h2>Criteria</h2>

However, there is one weakness about this pattern.  How do you specify your query criteria.  In the previous example we still need some way to specify exactly which user we want to run the query for.  I don't know the best way to overcome this issue.  best solution I have come up with so far is to have a super criteria object, that holds all the possible query criteria in one structure.  An example of this would be: 

{% highlight csharp %}
public struct QueryCriteria
{
	public UserCriteria User { get; private set; }
	public QuestionCriteria Question { get; private set; }

	public struct UserCriteria
	{
		public int? ByIdNumber { get; set; }
		public string ByName { get; set; }
	}
	
	public struct QuestionCriteria
	{
		public int? ByIdNumber { get; set; }
		public string ByQuestioningUserId { get; set; }
	}
}
{% endhighlight %}

This structure would be created, the requested criteria would be set, and then it would be passed into the query's Execute() method.  The individual queries would then look for any criteria being set that's relevant to them and retrieve the data using that criteria.  If no criteria was set that the query is written to support it will throw an exception.  One good thing about this method is that you now have one place where all query criteria can be found, which can give you a good reference for how queries are done in the system.

<h2>Final Thoughts</h2>

This is somewhat similar to [a previous post I made]({{ site.baseurl }}{% post_url 2011-05-25-allowing-eager-loading-in-business-logic-via-view-models %}), but after writing that post I realized the solution presented in that previous post was convoluted and needlessly complicated.  This seems like a much overall better, and simpler, solution to allow many complicated queries in an organized fashion.