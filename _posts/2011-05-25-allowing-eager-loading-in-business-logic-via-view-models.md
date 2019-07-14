---
layout: post
title:  "Allowing Eager Loading In Business Logic via View Models"
---

Today I am going to write a post about a library I have been thinking about for the past week.  If I end up actually developing it I plan to make it open source as I think it solves an issue that a lot of systems may encounter in regards to eager loading database entities, and dealing with a separation of the data entities from the domain model.  I am going to use this post to bring all of my thoughts together to make sure what's in my head actually makes sense, as well as to get any feedback (if this is actually read by anyone but me)

<h2>The Problem</h2>

My web application is currently using the Entity Framework 4.1 CodeFirst system.  The original issue came about when I was looking at document-oriented databases and the possibility of switching my backend to RavenDB.  It wasn't a serious idea, as it would be a huge hassle to convert the production data from sql server to a document oriented database for little benefit (at this time at least).  However, it did get me to realize that even though my architecture allows me to change database backends and ORMs pretty painlessly, it does not allow me to change my data model without changing my business entity model to match.  This is mostly due to my use of Linq to specify queries in my business layer.

My thoughts then went to switching to use a repository pattern for data access.  The repository pattern would allow me to more easily organize my queries, abstract away data access from the business layer, and would mean that my data model could be completely different from my business entity model and changes can be made to one without affecting the other.  Unfortunately, the repository pattern has a big issue when it comes to eager loading in that it usually ends up with many repeating methods, such as: 

{% highlight csharp %}
public class EFUserRepository : IUserRepository
{
    public User GetUserById(int id) { }
    public User GetUserByIdWithContacts(int id) { }
    public User GetUserByIdWithContactsWithCompanies(int id) { }
    ....
}
{% endhighlight %}

I have not found a better solution to handle eager loading with the repository pattern and the previous examples has too much repeating code for my liking.  

<h2>The VMQA (View Model Querying Assistant) Library</h2>

After thinking about this problem for a bit, I later realized that what I ultimately want is to pass a view model data structure into a repository query method, for the repository to come up with (and execute) an eager loading strategy based on the view model, and then to map all the properties of the view model with the data retrieved from the database. To accomplish this I want to rewrite my repository methods to look something like:

{% highlight csharp %}
public class EFUserRepository : IUserRepository
{
    public T GetUserById<T>(int id) where T : class 
    { 
        // Form the query
        var query = _context.Users.Where(x => x.Id == id);

        // Determine the eager loading strategy
        var includeStrings = vmqa.GetPropertiesToEagerLoadForClass(T).ConvertToEFStrings();
        foreach (string inc in includeStrings)
            query.Include(inc);

        // Execute the query and map results into the specified data structure
        var user = query.First();

        // Return an instance of the view model with the data filled in from the user
        return vmqa.MapEntityToViewModel<User, T>(user);
    }
}
{% endhighlight %}

The design I have floating around in my head to accomplish this has two parts.  

<h3>Eager Loading</h3>

The first part is coming up with an eager loading strategy based on the specified data structure, a data structure which the library has no previous knowledge of.  I envision this working by building a map of how all the data entities fit together with an interface that is similar to how configuring EF CodeFirst models is done.  An example mapping would be:

{% highlight csharp %}
public void MapEntities()
{
    vmqa.Entity<User>().HasMany(typeof(Contact), x => x.ContactsProperty);
    vmqa.Entity<Contact>().HasOne(typeof(Company), x => x.CompanyProperty);
}
{% endhighlight %}

This would tell the system that the <code>User</code> entity maps to <code>Contacts</code> via the <code>User.ContactsProperty</code>.

In my repository I would then call upon the VMQA library to look at the data structure and resolve the eager loading strategies.  To allow the VMQA library to understand how to map an arbitrary view model data structure to a data entity, I would use custom attributes that would decorate a class such as:

{% highlight csharp %}
[MapsToEntity(typeof(User))]
public class UserViewModel
{
    public int Id { get; set; }
    public string Name { get; set; }

    [MapsToEntity(typeof(Contact))]
    public IList Contacts { get; set; }
}
{% endhighlight %}

Using reflection, the <code>vmqa.GetPropertiesToEagerLoadForClass</code> should be able to figure out that the core entity maps to the User, that Contacts collection maps to the Contact.  Since it knows that the view model explicitly specifies a view model for the Contacts entity it can determine that the Contacts property needs to be eager loaded, and will then return a data structure containing the <code>Contact</code> type and the name of the property it maps to from the <code>User</code> entity.  

It should be easy to allow this system to allow complex mappings, so if the user view model has a property that maps to a collection of companies, the system can automatically find a path from <code>User</code> to  for eager loading.

The <code>ConvertToEFStrings()</code> extension method would then convert that data structure into a collection of strings that Entity Framework can use for eager loading.  This would be an extension method so that the library is not restricted to just Entity Framework, and can be used for any ORM or database system.

<h3>Entity Mapping</h3>

The second part of the whole system deals with automatically mapping the returned data entities into the view model.  This seems like a natural extension of the library to eager loading to me, as if you are passing the view model into the repository to describe what data you want to retrieve, it should automatically copy that data into the view model for your usage.  No matter what, the developer is going to want to do this anyway and it seem sensible to automatically do this with VMQA, since it already has knowledge about the relationships.

My first version of the library would probably do a straight name for name mapping, so that <code>UserViewModel.Name</code> maps to <code>User.Name</code>.  Eventually I would probably add property mapping attributes so more complex mappings can be performed.

<h2>Conclusion</h2>

After finally getting this design written down, I feel more confident than ever that this would be an extremely valuable library to have, and that it actually seems relatively straight-forward to create.  Now I just need the time!