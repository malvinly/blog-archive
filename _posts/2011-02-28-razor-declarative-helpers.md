---
layout: post
title: "Razor declarative helpers"
date: 2011-02-28 19:40:50 +0000
categories: ["Programming"]
tags: ["ASP.NET", "C#"]
original_url: https://nivlam.wordpress.com/2011/02/28/razor-declarative-helpers/
---

I’m currently converting an old ASP.NET MVC 2 project to MVC 3. In my MVC 2 project, I created HTML helpers to generate small bits of HTML.

{% raw %}
```csharp
public static HtmlString SampleHtmlHelper(this HtmlHelper helper, string input)
{
    string result = "<div>";
    result += "<p>Current time: " + DateTime.Today.ToShortDateString() + "</p>";
    result += "<p>Input: " + input + "</p>";
    result += "</div>";
     
    return new HtmlString(result);
}
```
{% endraw %}

These helper classes would reside in a Helpers folder that I create.

![]({{ site.baseurl }}/assets/images/2011/02/samplehelper.png)

To use the helper, I’d call it in a view like this:

{% raw %}
```csharp
<h2>Demo</h2>
 
@Html.SampleHtmlHelper("hello world")]
```
{% endraw %}

ASP.NET MVC 3 with the Razor view engine introduces a new way of creating declarative helpers that allows me to take advantage of the Razor syntax instead of appending strings together.

{% raw %}
```csharp
@helper SampleHelper(string input)
{
    <div>
        <p>Current date: @DateTime.Today.ToShortDateString()</p>
        <p>Input: @input</p>
    </div>
}
```
{% endraw %}

This helper would be placed in a view page in the `App_Code` folder.

![]({{ site.baseurl }}/assets/images/2011/02/demohelper.png)

I can treat each helper as a static method based on the view page name:

{% raw %}
```csharp
<h2>Demo</h2>
 
@Demo.SampleHelper("hello world")
```
{% endraw %}

I can convert most of my older HTML helpers to declarative helpers, but I’ve come across cases where I needed to continue creating helpers in code. I find that helpers that contain a lot of logic do not fit well with the Razor syntax.

{% raw %}
```csharp
public static HtmlString SampleHtmlHelper<TModel, TReturn>(this HtmlHelper<TModel> helper, Expression<Func<TModel, TReturn>> expression)
    where TModel : class
{
    MemberExpression memberExpression = GetMemberExpression(expression);
    string name = memberExpression == null ? null : memberExpression.Member.Name;
 
    Func<TModel, TReturn> function = expression.Compile();
    TReturn value = function(helper.ViewData.Model);
 
    string result = "<div>";
    result += "<p>" + name + ": " + value + "</p>";
    result += "</div>";
 
    return new HtmlString(result);
}
```
{% endraw %}

Being able to leverage the Razor syntax for helpers is a real bonus. I don’t see a need to use partial views anymore. When I needed to decide whether to use a helper or a partial view, it would be determined by how much markup I needed to generate. I would typically use a HTML helper if the amount of markup wasn’t a burden or if it contained logic. But with the new declarative helpers, this isn’t an issue anymore. Are there any benefits that a partial view has over declarative helpers?
