---
layout: post
title: "Forcing classes to implement ToString()"
date: 2012-02-07 01:31:51 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2012/02/06/forcing-classes-to-implement-tostring/
---

There are situations where it would be useful to force a class that implements an interface to override the `ToString()` method.

{% raw %}
```csharp
public interface ITask
{
	void Execute();
	string ToString();
}
```
{% endraw %}

In this case, I can execute a list of tasks and log any exceptions that occur. The `ToString()` method can provide a way to log information about the state or progress of a task when an exception occurs. The exception and stack trace may not be able to provide enough information in all situations.

{% raw %}
```csharp
foreach (ITask task in tasks)
{
	try
	{
		task.Execute();
	}
	catch (Exception e)
	{
		Log.Error("Error occurred: " + task.ToString(), e);
	}
}
```
{% endraw %}

Unfortunately, there isn't a way to force a class to override the `ToString()` method, even if the method signature is present on the interface. Instead, we can use abstract classes to enforce this by using the `abstract` and `override` modifiers together:

{% raw %}
```csharp
public abstract class Task
{
	public abstract void Execute();
	public abstract override string ToString();
}
```
{% endraw %}

Any classes that derive from `Task` will be required to override the `ToString()` method.
