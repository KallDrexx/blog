---
layout: post
title:  "My Adventures With RavenDB - Getting Distinct List Items"
---

I decided to play around with using the <a href="http://ravendb.net/">RavenDB</a> database system.  I wanted to see how fast it would take me to get a Raven database up and running.  I was very impressed with how easy it was, and for the most part it was just a matter of storing my records and using Linq to query for them.

<h2>No Select Manys</h2>

The only issue I came across was the fact that the Linq implementation does not implement the SelectMany() method.  From discussions, this is due to the linq queries being done against Lucene, and since Lucene data are stored flat it is impossible to look for data inside a list.  

The query I was trying to implement dealt with the following two data structures:

{% highlight csharp %}
public class LogRecord
{
    public Guid SessionId { get; set; }
    public IList<LogField> Fields { get; set; }
    public int RecordNumber { get; set; }
}

public class LogField
{
    public string FieldName { get; set; }
    public string StringValue { get; set; }
    public DateTime? DateValue { get; set; }
}
{% endhighlight %}

What I needed to do was to retrieve a distinct list of all FieldName values for a given SessionId value.  Normally with Linq I would use the following code:

{% highlight csharp %}
return ravenSession.Query<LogRecord>()
		.Where(x => x.SessionId == sessionId)
		.SelectMany(x => x.Fields)
		.Select(x => x.FieldName)
		.Distinct()
		.ToList();
{% endhighlight %}

This fails because SelectMany() is not supported by Raven.  

After doing some research it turns out that I needed to use Raven <a href="http://ayende.com/blog/4435/map-reduce-a-visual-explanation">Map/Reduce indexes</a>.  The reason Raven indexes work is because they run prior to Raven putting the data into Lucene, thus it can run the SelectMany() on the objects itself rather than just the data stored in Lucene.  

So in order to do this I coded the following index:

{% highlight csharp %}
 public class LogRecord_LogFieldNamesIndex : AbstractIndexCreationTask<LogRecord, LogSessionFieldNames>
{
    public LogRecord_LogFieldNamesIndex()
    {
        Map = records => from record in records
                            from field in record.Fields
                            select new
                            {
                                SessionId = record.SessionId,
                                FieldName = field.FieldName
                            };

        Reduce = results => from result in results
                            group result by new { result.SessionId, result.FieldName } into g
                            select new
                            {
                                SessionId = g.Key.SessionId,
                                FieldName = g.Key.FieldName
                            };
    }
    }
{% endhighlight %}

The query I used to access the list of field names now became:

{% highlight csharp %}
return session.Query<LogSessionFieldNames, LogRecord_LogFieldNamesIndex>()
			  .Where(x => x.SessionId == sessionId)
			  .Select(x => x.FieldName)
			  .Customize(x => x.WaitForNonStaleResultsAsOfNow())
			  .ToList();
{% endhighlight %}

Hope this helps someone!