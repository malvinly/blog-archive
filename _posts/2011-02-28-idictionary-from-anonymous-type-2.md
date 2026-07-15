---
layout: post
title: "IDictionary from anonymous type"
date: 2011-02-28 19:09:52 +0000
categories: ["Programming"]
tags: ["C#"]
original_url: https://nivlam.wordpress.com/2011/02/28/idictionary-from-anonymous-type-2/
---

In ASP.NET MVC, we can specify what attributes appear on our HTML elements by passing an anonymous type as an argument.

{% raw %}
```csharp
@Html.TextBox("my-textbox", "hello world", new { @class = "css-class", custom = "custom" })
```
{% endraw %}

In this example, we’re specifying that the `input` element will have a `class` attribute and a `custom` attribute when rendered.

Inspired by this, I needed a way to convert an anonymous type to an `IDictionary<string, object>`. I created this extension method that allows an object to be converted to a dictionary.

{% raw %}
```csharp
public static class ExtensionMethods
{
    public static IDictionary<string, object> ToDictionary(this object data)
    {
        BindingFlags publicAttributes = BindingFlags.Public | BindingFlags.Instance;
        Dictionary<string, object> dictionary = new Dictionary<string, object>();

        foreach (PropertyInfo property in data.GetType().GetProperties(publicAttributes))
        {
            if (property.CanRead)
                dictionary.Add(property.Name, property.GetValue(data, null));
        }

        return dictionary;
    }
}
```
{% endraw %}

This extends the `object` type and adds a `ToDictionary` extension method.

{% raw %}
```csharp
var anonymous = new { First = "John", Last = "Doe" };
IDictionary<string, object> dictionary = anonymous.ToDictionary();
```
{% endraw %}

Alternately, if we’re working in an ASP.NET MVC project, we can use `RouteValueDictionary` to convert an anonymous type to a dictionary.

{% raw %}
```csharp
var anonymous = new { First = "John", Last = "Doe" };
RouteValueDictionary dictionary = new RouteValueDictionary(anonymous);

string firstName = dictionary["First"].ToString();
string lastName = dictionary["Last"].ToString();
```
{% endraw %}
