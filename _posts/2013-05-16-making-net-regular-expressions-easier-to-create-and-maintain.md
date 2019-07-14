---
layout: post
title:  "Making .Net Regular Expressions Easier To Create and Maintain"
---

Recently I had the idea of a log parsing and analysis system, that would allow users (via a web interface) to define their own regular expressions to parse event logs in different ways.  The main purpose of this is for trend analysis.  <a href="http://logstash.net/">Logstash</a> is an open source project that does something similar, but it relies on configuration files for specifying regular expressions (and doesn't have other unrelated features I am envisioning).

One issue I have with regular expressions is they can be very hard to create and maintain as they increase in complexity.  This difficulty works against you if you are trying to create a system where different people with different qualifications should be able to define how certain logs are parsed.

While looking at Logstash, I was led to a project that it uses called <a href="https://code.google.com/p/semicomplete/wiki/Grok">Grok</a>, which essentially allows you to define aliases for regular expressions and even chain aliases together for more complicated regular expressions.  For example, if you needed a regular expression that included checking for an IP address, you can write <code>%{IP}</code> instead of <code>(?&lt;![0-9])(?:(?:25[0-5]|2[0-4][0-9]|[0-1]?[0-9]{1,2})[.](?:25[0-5]|2[0-4][0-9]|[0-1]?[0-9]{1,2})[.](?:25[0-5]|2[0-4][0-9]|[0-1]?[0-9]{1,2})[.](?:25[0-5]|2[0-4][0-9]|[0-1]?[0-9]{1,2}))(?![0-9])</code>.  This makes regular expressions <strong><em>much</em></strong> easier to create and read at later on.  

The problem with Grok is that it is written in C as a stand alone application.  This makes it hard to use in .Net applications without calling out to the shell, and even then it is cumbersome to use with dynamic aliases due to having to reconfigure config files on the fly.

For those reasons, I created an open source library I called <a href="https://github.com/KallDrexx/RapidRegex">RapidRegex</a>.  The first part of this library is the freedom to generate regular expression aliases by creating an instance of the <code>RegexAlias</code> class.  This class is used to give you full flexibility on how you store and edit regular expression aliases in any way your application sees fit, whether it's in a web form or in a flat file.  It does however come with a class that helps form <code>RegexAlias</code> structures from basic configuration files.

As a simple example, let's look at the regular expression outlined earlier for IP addresses.  With RapidRegex, you can create an alias for it and later convert aliased regular expressions into .net regular expressions.  An example of this can be seen with the following code:

{% highlight csharp %}
var alias = new RegexAlias
{
    Name = "IPAddress",
    RegexPattern = @"\b(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\b"
};

var resolver = new RegexAliasResolver(new[] { alias });

const string pattern = "connection from %{IPAddress}";
var regexPattern = resolver.ResolveToRegex(pattern);
// Resolved pattern becomes "connection from \b(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\b"
{% endhighlight %}

What makes this even more powerful is the fact that you can chain aliases together.  For instance, an IPv4 address can be defined as 4 sets of valid IP address bytes.  So we can accomplish the same above with:

{% highlight csharp %}
var alias = new RegexAlias
{
    Name = "IPDigit",
    RegexPattern = @"(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)"
};

var alias2 = new RegexAlias
{
    Name = "IPAddress",
    RegexPattern = @"%{IPDigit}\.%{IPDigit}\.%{IPDigit}\.%{IPDigit}"
};

var resolver = new RegexAliasResolver(new[] { alias, alias2 });

const string pattern = "connection from %{IPAddress}";
var regexPattern = resolver.ResolveToRegex(pattern);

// Resolve the pattern into becomes "connection from \b(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\b"
{% endhighlight %}

The project can be found on <a href="https://github.com/KallDrexx/RapidRegex">Github</a> and is licensed under the LGPL license.