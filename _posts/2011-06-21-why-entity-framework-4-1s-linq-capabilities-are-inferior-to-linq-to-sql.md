---
layout: post
title:  "Why Entity Framework 4.1's Linq capabilities are inferior to Linq-to-Sql"
---

I have a small-ish website at work that contains several tools we use internally.  I originally coded the database layer using Linq-to-Sql, as it was too small to be worth the learning curve of NHibernate and Entity Framework version 4 was not out at the time.  

However, lately database schema changes have been harder to incorporate into the Linq-to-Sql edmx, mostly due to the naming differences between database tables and relationships and their C# class names.  I decided to convert my database layer to Entity Framework 4.1 CodeFirst based on an existing database, as I use EF CodeFirst in a current project and already familiar with it.  

The process of converting my Linq-to-sql was *supposed* to be a simple matter of replacing my calls to my L2S data context to my EF DbContext, and allow Linq to take care of the rest.  Unfortunately, this is not the case due to Entity Framework's Linq capabilities being extremely limited.

<strong>Very Limited Function Support</strong>

In Linq-to-Sql it was common for me to perform transformations into a view model or an anonymous data type right in the linq.  Two examples of this in my code are:

{% highlight csharp %}
var client = (from c in _context.Clients
                where c.id == id
                select ClientViewModel.ConvertFromEntity(c)).First();

var clients = (from c in _context.Clients
                orderby c.name ascending
                select new
                {
                    id = c.id,
                    name = c.name,
                    versionString = Utils.GetVersionString(c.ProdVersion),
                    versionName = c.ProdVersion.name,
                    date = c.prod_deploy_date.ToString()
                })
                .ToList();
{% endhighlight %}

These fail when run in Entity Framework with NotSupportedExceptions.  The first one fails because it claims EF has no idea how to deal with ClientViewModel.ConvertFromEntity().  This would make sense, as this is a custom method, except this works perfectly in Linq-to-Sql.  The 2nd query fails not only for Utils.GetVersionString(), but it also fails because EF has no idea how to handle the ToString() off of a DateTime, even though this is all core functionality.

In order to fix this, I must return the results from the database and locally do the transformations, such as:

{% highlight csharp %}
var clients = _context.Clients.OrderBy(x => x.name)
    .ToList()
    .Select(x => new
    {
        id = c.id,
        name = c.name,
        versionString = Utils.GetVersionString(c.ProdVersion),
        versionName = c.ProdVersion.name,
        date = c.prod_deploy_date.ToString()
    })
    .ToList();
{% endhighlight %}

<strong>No More Entity to Entity Direct Comparisons</strong>

In Linq-to-Sql I could compare one entity directly to another in the query.  So for example the following line of code worked in L2S:

{% highlight csharp %}
context.TfsWorkItemTags.Where(x => x.TfsWorkItem == TfsWorkItemEntity).ToList();
{% endhighlight %}

This fails in Entity Framework and throws an exception because it can't figure out how to compare these in Sql.  Instead I had to change it to explicitly check on the ID values themselves, such as:

{% highlight csharp %}
context.TfsWorkItemTags.Where(x => x.TfsWorkItem.id == tfsWorkItemEntity.id).ToList();
{% endhighlight %}

It's a minor change, but I find it annoying that EF isn't smart enough to figure out how to compare entities directly, especially when it has full knowledge of how the entities are mapped and designed.  Yes, I could have gone straight through using TfsWorkitemEntity.Tags, but this is a simple example to illustrate lost functionality.

<strong>Cannot Use Arrays In Queries</strong>

This issue really caught me by surprise, and I don't understand why this was omitted.  In my database versions consist of 4 parts: major, minor, build, and revision numbers, and is usually represented in string form as AA.BB.CC.DD.  I have a utility method that converts the string into an array of ints, which I then used in my Linq-to-Sql query:

{% highlight csharp %}
int[] ver = Utils.GetVersionNumbersFromString(versionString);
return context.ReleaseVersions.Any(x => x.major_version == ver[0] && x.minor_version == ver[1]
                                    && x.build_version == ver[2] && x.revision_version == ver[3]);
{% endhighlight %}

Under L2S, this query works fine, but (as the common theme in this post) fails in Entity Framework 4.1 with a NotSupportedException with the message "The LINQ expression node type 'ArrayIndex' is not supported in LINQ to Entities.".  

In order to fix this I had to split my version into individual ints instead. 

<strong>Dealing With Date Arithmetic</strong>

My final issue I have come across is probably the most annoying, mostly because I can't find a good solution except to do most of the work on the client side.  In my database I store test requests, and those requests can be scheduled to run at a certain time on a daily basis.  This scheduled time is stored in a time(7) field in the database.  When I query for any outstanding test requests one of the criteria in the query is to make sure that the current date and time is greater than today's date at the scheduled time.  In L2S I had:

{% highlight csharp %}
var reqs = _context.TestRequests.Where(x => DateTime.Now > (DateTime.Now.Date + x.scheduled_time.Value)).ToList();
{% endhighlight %}

This fails in Entity Framework for 2 reasons.  The first is that DateTime.Now.Date isn't supported, so i had to create a variable to hold that and use that in the query.

The 2nd issue is then that EF can't make a query out of the current date added to a specific time value.  This causes an ArgumentException with the message "DbArithmeticExpression arguments must have a numeric common type.".  

I have not found a way to do this in EF, and instead had to resort to pulling a list of all TestRequest entities from the database, and locally pull out only the ones that fit that criteria.

<strong>Conclusion</strong>

I am utterly baffled by how limited Entity Framework is in it's Linq abilities.  While I will not go back to L2S for my internal tool I am definitely having second thoughts of using EF 4.1 in my more complex personal projects.  It definitely seems that Microsoft is going more for feature count in their frameworks rather than functional coverage lately.