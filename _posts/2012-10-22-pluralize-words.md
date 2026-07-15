---
layout: post
title: "Pluralize words"
date: 2012-10-22 02:14:15 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2012/10/21/pluralize-words/
---

Whenever I've needed the plural form of a word, I've always relied on the [`Inflector`](https://github.com/castleproject/ActiveRecord/blob/master/src/Castle.ActiveRecord/Framework/Internal/Inflector.cs) class from the Castle ActiveRecord project.

{% raw %}
```csharp
string plural = Inflector.Pluralize("planet");
Console.WriteLine (plural);
```
{% endraw %}

Since .NET 4.0, Microsoft has added their own [`PluralizationService`](http://msdn.microsoft.com/en-us/library/system.data.entity.design.pluralizationservices.pluralizationservice%28v=vs.100%29.aspx) class to pluralize words. Microsoft's `PluralizationService` class is actually used in the Entity Framework tools to generate singular or plural entity names.

{% raw %}
```csharp
var service = PluralizationService
	.CreateService(CultureInfo.GetCultureInfo("en-us"));

string plural = service.Pluralize("planet");
Console.WriteLine(plural);
```
{% endraw %}

There are cases when both classes do not generate the same plural form of a word. For example, attempting to pluralize the word "virus" generates different results. `Inflector` will return "viri" while the `PluralizationService` will return "virus". Neither generated the more common ["viruses"](http://en.wikipedia.org/wiki/Plural_form_of_words_ending_in_-us#Virus).
