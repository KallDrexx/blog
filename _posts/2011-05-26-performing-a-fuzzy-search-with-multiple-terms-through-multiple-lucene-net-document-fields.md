---
layout: post
title:  "Performing a fuzzy search with multiple terms through multiple Lucene.Net document fields"
---

Recently I have been trying to implement Lucene.Net into my web application to allow users to search through their data.  My core requirements were that the search allows minor mis-spellings and that search terms can can be found across different fields.  The code to accomplish this wasn't immediately obvious, and it took me a few days and a few questions posted to StackOverflow to finally figure out how to do it.      Therefore, I thought it would be beneficial to others to show how to accomplish this.

To illustrate my scenario let's assume I am searching for data indexed based on the following class:

{% highlight csharp %}
public class Contact
{
    public string FirstName { get; set};
    public string LastName { get; set;};
}
{% endhighlight %}

When I index this structure in Lucene I decided to store the <code>FirstName</code> and <code>LastName</code> properties into separate fields in the Lucene document.  This is to allow more advanced searching at a later time.  

So for this scenario I want to allow a user to search for a contact with the name "Jon Doe", but also allow the user to find the contact by using the search strings "John Doe", "Jon", or "Doe".  

The first thing that has to be done is to use a <code>MultiFieldQueryParser()</code> which allows a Lucene.net to search for the search terms in all of the specified fields of a document.  The MultiFieldQueryParser can be used by the following code:

{% highlight csharp %}
public SearchResults Search(string searchString)
{
    // Define fields in the document to search through
    string[] searchableFields = new string[] { "FirstName", "LastName" };

    // Generate the parser
    var parser = new MultiFieldQueryParser(Lucene.Net.Util.Version.LUCENE_29, searchfields, new                                                 StandardAnalyzer(Lucene.Net.Util.Version.LUCENE_29));
    parser.SetDefaultOperator(QueryParser.Operator.AND);

    // Perform the search
    var query = parser.Parse(searchString);
    var directory = FSDirectory.Open(new DirectoryInfo(LuceneIndexBaseDirectory));
    var searcher = new IndexSearcher(directory, true);
    var hits = searcher.Search(query, MAX_RESULTS);

    // Return results
}
{% endhighlight %}

The parser will split up all the terms of the search string and look for each term individually in all of the specified fields (I used an AND comparison with <code>QueryParser.Operator.AND</code> so that all search terms must be found for a document to be found).  This code will allow me to perform successful searches with a search string of "Jon", "Jon Doe", and "Doe" but will <strong><em>not</em></strong> allow me to search for "John" as this method does not contain any code to allow fuzzy searches.

Implementing fuzzy search turned out to be the confusing part.  The Lucene documentation's only explanation for how to perform a fuzzy search is to add a tilde(~) to the end of a search term.  With this knowledge, my first attempt was to add a tilde to the end of each word in the search query string.  Unfortunately, it turns out that when Lucene is given multiple words in a search string and each word has a tilde at the end, a fuzzy search is only performed for the last word in the query, all others seem to be ignored and must be exact matches to get a hit.  

After asking around it turns out that in order to allow each and every word to be non-exactly matched in the search you have to create a separate Lucene query for each term in the search string, and combine all the queries together using the <code>BooleanQuery</code> class.  My implementation of this is:

{% highlight csharp %}
public SearchResults Search(string searchString)
{
    // Setup the fields to search through
    string[] searchfields = new string[] { "FirstName", "LastName" };

    // Build our booleanquery that will be a combination of all the queries for each individual search term
    var finalQuery = new BooleanQuery();
    var parser = new MultiFieldQueryParser(Lucene.Net.Util.Version.LUCENE_29, searchfields, CreateAnalyzer());

    // Split the search string into separate search terms by word
    string[] terms = searchString.Split(new[] { " " }, StringSplitOptions.RemoveEmptyEntries);
    foreach (string term in terms)
        finalQuery.Add(parser.Parse(term.Replace("~", "") + "~"), BooleanClause.Occur.MUST);

    // Perform the search
    var directory = FSDirectory.Open(new DirectoryInfo(LuceneIndexBaseDirectory));
    var searcher = new IndexSearcher(directory, true);
    var hits = searcher.Search(finalQuery, MAX_RESULTS);
}
{% endhighlight %}

The <code>BooleanClause.Occur.MUST</code> ensures that all term queries must produce a match in order for the document to be a hit for the search.  

The purpose of the <code>term.Replace("~", "") + "~"</code> code is to add a tilde to the end of the search term so Lucene knows to perform a fuzzy search with this term.  We must remove any tildes the user entered in the search string to prevent an exception that will occur if a search term ends with two tildes (e.g. "John~~").

This will successfully allow you to perform a fuzzy search with multiple terms across multiple document fields.  The algorithm does eventually need to be improved to not split out search terms that are inside of quotations (to signify a search for a phrase) but I hope this helps others figure out fuzzy searching more quickly than it took me.