---
layout: post
title:  "Mimicking Html.BeginForm() to reduce html div duplication in Asp.Net MVC sites"
---

Recently I have been trying to find ways to reduce how much HTML I have to duplicate in my views and how I have to remember what css classes to give each set of divs.  The problem comes with that the HTML that my views require don't fit in the main layout, due to the fact that content still comes after it and some elements are optional, and they don't fit well in partial views due to how customized the HTML inside the area is.  

An example of the HTML I was dealing with, here's an example of one of my views

{% highlight csharp %}
<div class="grid1 floatLeft"> 
    <div class="lineSeperater"> 
        <div class="pageInfoBox"> 
            @using (Html.BeginForm(MVC.JobSearch.Edit())) 
            {
                @Html.HiddenFor(x => x.Id)
                
                <div class="grid3 marginBottom_10 marginAuto floatLeft"> 
                    <h3 class="floatLeft">@(isNewJobSearch ? "Start Job Search" : "Edit Job Search")</h3> 
                </div> 
                
                <div class="grid3 marginBottom_10 marginAuto floatLeft">
                    <div class="floatLeft">
                        <p>
                            Displayed page summary
                        </p>    
                    </div>
                </div>
                
                <div class="grid3 marginBottom_10 marginAuto floatleft">
                    <div class="floatLeft infoSpan">
                        @Html.ValidationSummary()
                    </div>
                </div>
                
                <div class="grid3 marginBottom_10 floatLeft"> 
                    <div class="floatLeft"><p class="greyHighlight">Title:</p>
                        <div class="infoSpan">@Html.TextBoxFor(x => x.Name, new { @class = "info" })</div>
                    </div> 
                </div> 

                <div class="grid3 marginBottom_10 floatLeft"> 
                    <div class="floatLeft"><p class="greyHighlight">Description:</p>
                        <div class="infoSpan">@Html.TextAreaFor(x => x.Description, new { @class = "textAreaInfo" })</div>
                    </div> 
                </div> 

                <div class="grid3 marginBottom_20 floatLeft"> 
                    <div class="submitBTN "><input type="submit" value="Save" /></div>                    
                </div> 
            }

            <div class="clear"></div> 
        </div> 
    </div> 
</div> 

<!-- More HTML Here -->
{% endhighlight %}

I started thinking of how I can increase my code re-use to make this easier to develop and maintain.  While looking over the view my eyes gravitated towards the Html.BeginForm() and I realized the most logical idea was to utilize using statements.  So after looking at the implementation of Html.BeginForm() for guidance (thanks to <a href="http://www.jetbrains.com/decompiler/" title="dotPeek decompiler">dotPeek</a>) I came up with the following class to implement the first few divs automatically.

{% highlight csharp %}
    public class PageInfoBoxWriter : IDisposable
    {
        protected ViewContext _viewContext;
        protected bool _includesSeparator;

        public PageInfoBoxWriter(ViewContext context, bool includeSeparator)
        {
            if (context == null)
                throw new ArgumentNullException("context");

            _viewContext = context;
            _includesSeparator = includeSeparator;

            // Write the html
            _viewContext.Writer.Write("<div class=\"grid1 floatLeft\">");

            if (_includesSeparator) _viewContext.Writer.Write("<div class=\"lineSeperater\">");

            _viewContext.Writer.Write("<div class=\"pageInfoBox\">");

            return;
        }

        public void Dispose()
        {
            _viewContext.Writer.Write("<div class=\"clear\"></div></div></div>");
            if (_includesSeparator) _viewContext.Writer.Write("</div>");
        }
    }
{% endhighlight %}

I then created an html helper to use this class

{% highlight csharp %}
public static class LayoutHelpers
{
    public static PageInfoBoxWriter PageInfoBox(this HtmlHelper html, bool includeSeparator)
    {
        return new PageInfoBoxWriter(html.ViewContext, includeSeparator);
    }
}
{% endhighlight %}

This will write the beginning divs upon creation and the div end tags upon closing.  After following suit with the other elements of my page and creating more disposable layout classes and Html helpers, I now have the following view:

{% highlight csharp %}
@using (Html.PageInfoBox(false))
{
    using (Html.BeginForm(MVC.JobSearch.Edit())) 
    {
        @Html.HiddenFor(x => x.Id)

        using (Html.OuterRow())
        {
            <h3 class="floatLeft">@(isNewJobSearch ? "Start Job Search" : "Edit Job Search")</h3> 
        }

        using (Html.OuterRow())
        {
            <div class="floatLeft">
                <p>
                    Displayed page summary
                </p>    
            </div>
        }

        using (Html.OuterRow())
        {
            <div class="floatLeft infoSpan">
                @Html.ValidationSummary()
            </div>
        }

        using (Html.OuterRow())
        {
            Html.FormField("Title:", Html.TextBoxFor(x => x.Name, new { @class = "info" }));
        }

        using (Html.OuterRow())
        {
            Html.FormField("Description:", @Html.TextAreaFor(x => x.Description, new { @class = "textAreaInfo" }));
        }

        using (Html.OuterRow())
        {
            using(Html.FormButtonArea())
            {
                <input type="submit" value="Save" />
            }
        }
    }
}

<!-- other html here -->
{% endhighlight %}

Now I have a MUCH more maintainable view that even gives me intellisense support, so I don't have to worry about remembering how css classes are capitalized, how they are named, what order the need to be in, etc...  