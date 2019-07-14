---
layout: post
title:  "Beware of Asp.Net's Membership Provider Lifetime, Entity Framework Caching, and Dependency Injection"
---
I recently struggled while dealing with a bug I encountered in my Asp.Net MVC application, and I thought I would write about it to hopefully help someone else.  

I have a method in my business layer that users use to change their password.  After implementing unit tests to verify the method worked properly I implemented a call to it in my account controller.  Everything seemed perfect at first, as I was able to change my password successfully and even validate that the user entered their correct current password, which is required to perform a password change.  

However, when I logged out and tried to log back in using the new password the login attempt failed.  Yet when I used my previous password I was able to log in!  To make things even more confusing, since I require users to enter their current password to change their password I was able to confirm that the password change <strong>did</strong> actually take effect.  Finally, when I made a debugging change and re-ran the app, the new password worked when logging in!

A few irritating hours later, I finally figured out what was wrong.  It came down to the difference of lifetimes in my MVC application between different classes.  My AccountMembershipService class, which was mostly based on the default code that came with MVC, had the following constructor:

{% highlight csharp %}
public AccountMembershipService()
    : this(null)
{
}

public AccountMembershipService(MembershipProvider provider)
{
    _provider = provider ?? Membership.Provider;
}
{% endhighlight %}

The problem is that my application loads its service classes using Castle Windsor, and entity framework is loaded from Windsor with a per-web request lifetime.  Even though my custom membership provider was retrieving the database context from Windsor, Asp.Net creates the membership provider for the lifetime of the <strong>whole application</strong>.  

So what happened was that I would log in using the database context for the Membership Provider, but when I changed my password the service class would use a different database context.  When I changed my password, the password is correctly changed in the database (and the request's specific database context), but the Membership Provider is still using the original database context.  Since Entity Framework caches previously received entities, when I go back to log in entity framework receives the user entity out of the cache, <strong>not the database</strong>, and thus the old password is what will pass the Membership Provider's validation check.

The fix was to replace the previous code with:

{% highlight csharp %}
public AccountMembershipService(MembershipProvider provider)
{
    _provider = provider ?? new MyMembershipProvider();
}
{% endhighlight %}

This forces a new membership provider instance to be created for that web request, and thus guarantees that the database context will not be holding a stale user entity.

Hopefully this saves someone from the irritation and wasted time that I went through.

<hr>
<strong>Update:</strong> I have written [another post]({{ site.baseurl }}{% post_url 2011-09-28-asp-net-membership-providers-lifetime-considerations-part-2 %}) adding on to the issues I wrote about here.