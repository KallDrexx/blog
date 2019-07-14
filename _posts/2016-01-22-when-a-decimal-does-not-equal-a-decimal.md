---
layout: post
title:  "When a Decimal does not equal a Decimal"
---

It all started with a simple bug ticket in Trello that said "When I select 0.125 from the drop down, save, then enter the edit screen again the drop down is set to 0".

Seemed pretty simple. The clock said 5:45pm and I figured "this should be easy to knock out". An hour later I was pulling my hair out.
<h1>What?</h1>
Here's a good example of the offending code:

{% highlight csharp %}
public class FooRepository
{
	public decimal GetFooValue() 
	{
		return 0.1250m;
	}
}

public class FooModel
{
	public decimal Value { get; set; }
}

public class FooController
{
	public ActionResult Index()
	{
		var repository = new FooRepository();
		var model = new FooModel { Value = repository.GetFooValue() };
		return View(model);
	}
}
{% endhighlight %}

Then in the view:

{% highlight csharp %}
@Html.DropDownListFor(x =&gt; x.Value, new SelectList(new[] {0m, 0.125m, 0.25m}))
{% endhighlight %}

Every time I displayed this view the drop down was always on the first item and it was driving me nuts. After playing around in Linqpad I came across a clue:

{% highlight csharp %}0.250m.ToString(); // Outputs "0.250"{% endhighlight %}

I stared for a while and finally noticed the trailing zero in the displayed output. I then made sure I wasn't totally crazy so I tried:

{% highlight csharp %}
0.250m.ToString() == 0.25m.ToString() // outputs False
{% endhighlight %}

I looked in my real database where the value was coming from, and since it is a <code>decimal(18,4)</code> entity framework is bringing it back with 4 decimal places, which means it's including the trailing zero. Now it makes sense why Asp.net MVC's helpers can't figure out which item is selected, as it seems like a fair assumption that it is calling <code>ToString()</code> and doing comparisons based on that.

While trying to figure out a good solution I came across <a href="http://stackoverflow.com/a/7983330/231002">this StackOverflow answer</a> which had the following extension method to normalize a decimal (which removes any trailing zeros):

{% highlight csharp %}
public static decimal Normalize(this decimal value)
{
    return value/1.000000000000000000000000000000000m;
}
{% endhighlight %}

After normalizing my model's value, the select list worked perfectly as I expected and the correct values were picked in the drop down list.

Of course this is a clear hack based on implementation details and I reverted this fix.  There's no telling if or when a new version of the .Net framework may change this, and from what I've read a lot of the internal details of how decimals work are different in Mono, and this hack does not play nicely in Mono (supposedly in mono, <code>0.1250m.ToString()</code> does not display trailing zeros).

The proper way to resolve this situation is to force the drop down list to do a numerical equality to determine which item should be selected, by manually creating a <code>List</code> instead of using the <code>SelectList()</code> constructor.

<h1>Why?</h1>
So even though I knew what was going wrong, I was interested in why. So that stack overflow answer pointed me to the <a href="https://msdn.microsoft.com/en-us/library/system.decimal.getbits.aspx">MSDN documentation for the Decimal.GetBits() method</a>. That link contained specifications for how a decimal is actually constructed. Specifically that internally a decimal is made up of 4 integers, 3 representing the low, medium, and high bits in the value and the last integer containing information on the power of 10 exponent for the value. Those combined can give exact decimal values (within the specified range and decimal places allowed).

So to start with I tried the decimal of 1m and printed out the bit representation:

{% highlight csharp %}
new[] {
	Convert.ToString(decimal.GetBits(1m)[0], 2),
	Convert.ToString(decimal.GetBits(1m)[1], 2),
	Convert.ToString(decimal.GetBits(1m)[2], 2),
	Convert.ToString(decimal.GetBits(1m)[3], 2),
}

// 0000000000000000000000000000001
// 0000000000000000000000000000000
// 0000000000000000000000000000000
// 0000000000000000000000000000000
{% endhighlight %}

That didn't show much unexpected. It shows the low bits as containing just a 1 and all other bits containing zeros. Next I tried 1.0m:

{% highlight csharp %}
new[] {
	Convert.ToString(decimal.GetBits(1.0m)[0], 2),
	Convert.ToString(decimal.GetBits(1.0m)[1], 2),
	Convert.ToString(decimal.GetBits(1.0m)[2], 2),
	Convert.ToString(decimal.GetBits(1.0m)[3], 2),
}

// 0000000000000000000000000101000
// 0000000000000000000000000000000
// 0000000000000000000000000000000
// 0000000000000010000000000000000
{% endhighlight %}

This parses the core bits to:
lo: 10
mid: 0
high: 0

So how does it convert a 10 into a 1? Looking back at the MSDN documentation it says that the first 15 bits are always zero (as they are in this case) with the next seven bits being the exponent of the decimal. Bits 16-23 are 0000001 and the last bit is zero, giving us an exponential value of positive one. These mean that to get the final value we take the value of the low + mid + high bits combined (10) and divide them by 10 to the power of positive 1. This gives us a value of just 1,

If we look at our 0.125m example we get:

{% highlight csharp %}
new[] {
	Convert.ToString(decimal.GetBits(0.125m)[0], 2),
	Convert.ToString(decimal.GetBits(0.125m)[1], 2),
	Convert.ToString(decimal.GetBits(0.125m)[2], 2),
	Convert.ToString(decimal.GetBits(0.125m)[3], 2),
}

// 0000000000000000000000001111101
// 0000000000000000000000000000000
// 0000000000000000000000000000000
// 0000000000000110000000000000000
{% endhighlight %}

Like before, this is taking a value of 125 (125 + 0 + 0) and dividing it by 10 to the positive 3 exponent, which gives us 0.125. If we instead use 0.1250m we get:

{% highlight csharp %}
new[] {
	Convert.ToString(decimal.GetBits(0.1250m)[0], 2),
	Convert.ToString(decimal.GetBits(0.1250m)[1], 2),
	Convert.ToString(decimal.GetBits(0.1250m)[2], 2),
	Convert.ToString(decimal.GetBits(0.1250m)[3], 2),
}

// 0000000000000000000010011100010
// 0000000000000000000000000000000
// 0000000000000000000000000000000
// 0000000000001000000000000000000
{% endhighlight %}

This represents 1,250 (1,250 + 0 + 0) divided by 10 to the positive 4 exponent.

So now it's clear that when you create a new decimal it essentially keeps track of the exact representation it was originally invoked with, including zeros and is pieced together to it's fully realized value on a ToString() call.

