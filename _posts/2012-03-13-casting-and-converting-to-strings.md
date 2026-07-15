---
layout: post
title: "Casting and converting to strings"
date: 2012-03-13 05:42:07 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2012/03/13/casting-and-converting-to-strings/
---

One thing that makes me nervous while looking through a code base is the liberal use of the `ToString()` method everywhere. There are a couple different ways to convert/cast to a string:

Explicit cast:
{% raw %}
```csharp
string str = (string)someObj;
```
{% endraw %}

`ToString()` method:
{% raw %}
```csharp
string str = someObj.ToString();
```
{% endraw %}

`Convert.ToString()` method:
{% raw %}
```csharp
string str = Convert.ToString(someObj);
```
{% endraw %}

`as` operator:
{% raw %}
```csharp
string str = someObj as string;
```
{% endraw %}

Explicit casts should be used at times when you are sure that `someObj` is a string. It also doubles as an assertion since an `InvalidCastException` will be thrown at runtime if the cast fails. The result will be `null` if attempting to cast a `null` object.

The `ToString()` method returns the string representation of an object. If `someObj` is null, a `NullReferenceException` will be thrown.

The `Convert.ToString()` method behaves differently depending on the type of object passed to the method. When an `object` is passed to the method, it calls the `ToString()` method on the object:

{% raw %}
```csharp
public static string ToString(Object value) 
{
	return ToString(value, null);
} 

public static string ToString(Object value, IFormatProvider provider) 
{ 
	IConvertible ic = value as IConvertible; 
	
	if (ic != null)
		return ic.ToString(provider); 
		
	IFormattable formattable = value as IFormattable;
	
	if (formattable != null)
		return formattable.ToString(null, provider);
		
	return value == null ? String.Empty : value.ToString(); 
}
```
{% endraw %}

When a string is passed to the `Convert.ToString()`, it simply passes through:

{% raw %}
```csharp
public static String ToString(String value) 
{
	Contract.Ensures(Contract.Result<string>() == value);
	return value; 
}
```
{% endraw %}

This is important to remember because if a `null` object is passed to the `Convert.ToString()` method, an empty string is returned. However, if a `null` string is passed to the method, a `null` is returned.

The `as` operator provides a way to cast an object without throwing an exception if the cast fails. If a cast fails, the result will be `null`.
