---
layout: post
title:  "Asp.Net Membership Provider's Lifetime Considerations- Part 2"
---

Previously I made a post about [issues I encountered with the Asp.Net Membership Provider]({{ site.baseurl }}{% post_url 2011-09-26-beware-of-asp-nets-membership-provider-lifetime-entity-framework-caching-and-dependency-injection %}).  

My fix was a bit short-sighted as it only fixed the issue in one location, when the account controller tries to perform operations.  The problem with my solution is that it does not deal with all the areas where Asp.Net creates the membership provider itself, and keeps that instance running through the application lifetime.  

This meant that all calls to the membership provider (e.g. calling Membership.GetUser()) were all using the same database session, even across multiple web requests.  Not only did this cause hidden caching issues, this also mean that if the database connection was broken all calls to the membership provider would now exception out with the only way to fix it is to restart the Asp.Net instance.

In order to fix this I had to remove my database session creation out of my custom membership provider's constructor, and instead retrieve a new database session from my IoC system inside each method.  

This seems to not only fix my original issue, but several other "random" exceptions that I could not reproduce on a regular basis.   