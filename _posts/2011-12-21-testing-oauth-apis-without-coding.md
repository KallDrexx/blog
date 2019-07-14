---
layout: post
title:  "Testing OAuth APIs Without Coding"
---

While working with the LinkedIn API, I started becoming frustrated in testing my API calls.  The core reason was that I couldn't just form my URL in the web browser due to OAuth.  In order to test my API calls I would have to write code to perform the call, test it out, use the VS debugger to retrieve the result to make sure the XML it's returning is what I expect, etc..  It results in a lot of wasted time and I finally got fed up.

<h2>Introducing the ExtApi Tester</h2>

I developed a windows application to make testing API calls, especially API calls that require OAuth, to be done much simpler.  I call it the ExtApi Tester.

<img src="{{ site.url }}/assets/img/extapi-screenshot.png" alt="" title="ExtApi-Screenshot" />

The code can be found on <a href="https://github.com/KallDrexx/ExtApi">GitHub</a>.  I've already gotten some good use out of it, and hopefully it helps someone else with their dev experience.  

The code also includes an API for making it easier to call web API's from code, but it requires some further refinement to handle the DotNetOpenAuth authorization process. 