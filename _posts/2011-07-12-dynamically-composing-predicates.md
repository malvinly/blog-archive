---
layout: post
title: "Dynamically composing predicates"
date: 2011-07-12 12:39:36 +0000
categories: ["Programming"]
original_url: https://nivlam.wordpress.com/2011/07/12/dynamically-composing-predicates/
---

Typically when I'm using LINQ to filter collections, my conditions aren't very complex and usually contain a single boolean condition. For example, if I wanted to filter a collection of `WorkItems`, I'll use the `Where` extension method:

{% raw %}
```csharp
workItems.Where(x => x.Status == Status.Done);
```
{% endraw %}

I've come across a scenario where I needed to chain multiple "OR" expressions. It's easy to chain multiple "OR" expressions during compile time:

{% raw %}
```csharp
workItems.Where(x => x.Status == Status.Done || x.Status == Status.Processing);
```
{% endraw %}

However, this is difficult to do if the conditions are determined at runtime. In an application, I'm asking the user to select from a list of statuses. They may select one or more statuses.

![]({{ site.baseurl }}/assets/images/2011/07/checkboxes.png)

Since I still work a lot with stored procedures, I'll typically use dynamic SQL to generate a WHERE clause if these status were stored in a database. But with a collection, this isn't easy to do without some help.

After looking around, I stumbled across the [PredicateBuilder](http://www.albahari.com/nutshell/predicatebuilder.aspx "PredicateBuilder") class that ships as part of [LINQKit](http://www.albahari.com/nutshell/linqkit.aspx "LINQKit").

The `PredicateBuilder` allows me to easily chain multiple "AND" and "OR" conditions together at runtime. I can append an "OR" condition to the predicate if the respective checkbox is checked:

{% raw %}
```csharp
var predicate = PredicateBuilder.False<WorkItem>();

if (cbIdle.Checked)
    predicate = predicate.Or(x => x.Status == Status.Idle);

if (cbProcessing.Checked)
    predicate = predicate.Or(x => x.Status == Status.Processing);

if (cbDone.Checked)
    predicate = predicate.Or(x => x.Status == Status.Done);

workItems.Where(predicate.Compile());
```
{% endraw %}

The `PredicateBuilder` also allows me to nest predicates and create complex conditions. For example, if I wanted to create a predicate that is equivalent to the following:

{% raw %}
```csharp
workItems.Where(x => 
    x.Status == Status.Done &&
    (x.Description.Contains("report") || x.Description.Contains("summary"))
);
```
{% endraw %}

... I can create two predicates and append them together.

{% raw %}
```csharp
var description = PredicateBuilder.False<WorkItem>();
description = description.Or(x => x.Description.Contains("report"));
description = description.Or(x => x.Description.Contains("summary"));

var condition = PredicateBuilder.True<WorkItem>();
condition = condition.And(x => x.Status == Status.Done);
condition = condition.And(description);

workItems.Where(condition.Compile());
```
{% endraw %}
