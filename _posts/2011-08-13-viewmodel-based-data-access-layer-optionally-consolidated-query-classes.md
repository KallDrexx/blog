---
layout: post
title:  "A ViewModel Based Data Access Layer – Optionally Consolidated Query Classes"
---

After thinking upon my data access pattern outlined in <a href="http://wp.me/p1Ad8K-1u">my last post</a>, I came up with the following interface to use for queries:

{% highlight csharp %}
public interface IQuery<TViewModel, TCriteria>
{
	TViewModel Execute(TCriteria criteria);
}
{% endhighlight %}

I am very happy with this idea, as it seems to have multiple advantages over current data access patterns I have evaluated.

<h2>Optionally Consolidated Queries</h2>

One advantage that I see of this pattern is that it's trivial to combine similar queries into one class, while keeping the business layer ignorant of the grouping.

To show this let's use the example of <a href="http://www.stackoverflow.com">Stack Overflow</a>.  If you look at the homepage and your user's page you will notice that in these pages there are 2 queries that are retrieving data for a specific user, but each query returns different information.  The homepage only requires the user's username, reputation, and badge data.  However, when you view a user's page it needs that to query for that information as well as questions and answers related to the user.  Even though both queries deal with retrieving data for a specific user it would be inefficient to use the latter query for the homepage, as it would have to hit multiple tables when that data isn't used.

An example of creating an MVC controller that uses my ViewModel DAL would be:

{% highlight csharp %}
public class ExampleController : Controller
{
	protected IQuery<UserHomepageViewModel, UserByIdCriteria> _userHomepageQuery;
	protected IQuery<UserDashboardViewModel, UserByIdCriteria> _userDashboardQuery;
	protected int _currentUserId;
	
	public ExampleController(
		IQuery<UserHomepageViewModel, UserByIdCriteria> userHomepageQuery,
		IQuery<UserDashboardViewModel, UserByIdCriteria> userDashboardQuery)
	{
		_userHomepageQuery = userHomepageQuery;
		_userDashboardQuery = userDashboardQuery;
		_currentUserId = (int)Membership.GetUser().UserProviderKey;
	}
	
	public ActionResult Index()
	{
		var criteria = new UserByIdCriteria { UserId = _currentUserId };
		var model = _userHomepageQuery.Execute(criteria);
		return View(model);
	}
	
	public ActionResult UserDashboard()
	{
		var criteria = new UserByIdCriteria { UserId = _currentUserId };
		var model = _userDashboardQuery.Execute(criteria);
		return View(model);
	}	
}
{% endhighlight %}

As far as the programmer in charge of this controller is concerned, two completely separate classes are used for these queries.  However, the developer can save some effort by implementing these queries into one consolidated class.  For example:  

{% highlight csharp %}
public class UserByIdQueries : IQuery<UserHomepageViewModel, UserByIdCriteria>, IQuery<UserDashboardViewModel, UserByIdCriteria>
{
	protected DbContext _context;
	
	public UserByIdQueries(DbContext context)
	{
		_context = context;
	}
	
	public UserHomepageViewModel Execute(UserByIdCriteria criteria)
	{
		var user = GetUserById(criteria.UserId, false);
		return Mapper.Map<User, UserHomepageViewModel>(user);
	}
	
	public UserDashboardViewModel Execute(UserByIdCriteria criteria)
	{
		var user = GetUserById(criteria.UserId, true);
		return Mapper.Map<User, UserDashboardViewModel>(user);
	}
	
	protected User GetUserById(int id, bool includeDashboardData)
	{
		var query = _context.Users.Where(x => x.Id == id);
		
		if (includeDashboardData)
			query.Include(x => x.Questions).Include(x => x.Answers);
			
		return query.SingleOrDefault();
	}
}
{% endhighlight %}

To me, this gives the perfect balance of easily retrieving data from the DAL based on how I am going to use the data and still give me full flexibility on how I organize and create the DAL's implementation.