---
layout: post
title: ".NET 4.0 and first chance exceptions"
date: 2012-05-25 12:55:38 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2012/05/25/net-4-0-and-first-chance-exceptions/
---

.NET 4.0 introduced a new event that is raised for every exception that is thrown in an application domain. The [`FirstChanceException`](http://msdn.microsoft.com/en-us/library/system.appdomain.firstchanceexception.aspx) event is raised as soon as an exception is thrown. This includes any handled and unhandled exceptions.

{% raw %}
```csharp
static void Main(string[] args)
{
	AppDomain.CurrentDomain.FirstChanceException += (sender, eventArgs) =>
	{
		Console.WriteLine("First chance exception: " + eventArgs.Exception.Message);
	};

	try 
	{
		Console.WriteLine("Before exception.");
		throw new Exception("Some error.");
	}
	catch
	{
		Console.WriteLine("Handled.");
	}
}
```
{% endraw %}

This code will produce the output:

{% raw %}
```text
Before exception.
First chance exception: Some error.
Handled.
```
{% endraw %}

However, if an exception is thrown within the event handler, the `FirstChanceException` event will be raised recursively. The code below will cause a stack overflow:

{% raw %}
```csharp
static void Main(string[] args)
{
	AppDomain.CurrentDomain.FirstChanceException += (sender, eventArgs) =>
	{
		try
		{
			throw new Exception();
		}
		catch
		{
		}
	};

	try 
	{
		throw new Exception();
	}
	catch
	{
	}
}
```
{% endraw %}

This is important to remember if we use any third party libraries, such as a logging library like Log4Net, to log exceptions within the event handler. These third party libraries may throw an exception internally, which will cause the `FirstChanceException` event to be raised recursively. Even without the use of third party libraries, any code may unexpectedly thrown an exception, which will cause the event to be raised recursively and cause a stack overflow. The MSDN documentation mentions this fact:

> You must handle all exceptions that occur in the event handler for the FirstChanceException event. Otherwise, FirstChanceException is raised recursively. This could result in a stack overflow and termination of the application. We recommend that you implement event handlers for this event as constrained execution regions (CERs), to keep infrastructure-related exceptions such as out-of-memory or stack overflow from affecting the virtual machine while the exception notification is being processed.

Unfortunately, the documentation doesn't provide any code samples. I haven't been able to figure a way to prevent stack overflows without resorting to a somewhat non-elegant solution. What we can do is check to see if the event handler is found multiple times in the current stack trace. If it is, we can immediately return to prevent the event from being raised again.

{% raw %}
```csharp
static void Main(string[] args)
{
	AppDomain.CurrentDomain.FirstChanceException += (sender, eventArgs) =>
	{
		StackFrame[] frames = new StackTrace(1).GetFrames();
		MethodBase currentMethod = MethodBase.GetCurrentMethod();

		if (frames != null && frames.Any(x => x.GetMethod() == currentMethod))
			return;

		throw new Exception();
	};

	throw new Exception();
}
```
{% endraw %}

Instantiating [`StackTrace`](http://msdn.microsoft.com/en-us/library/wybch8k4.aspx) may throw an `ArgumentOutOfRangeException` if the parameter is negative, which is not what we're doing here. The [`GetFrames()`](http://msdn.microsoft.com/en-us/library/system.diagnostics.stacktrace.getframes.aspx) method may return `null`, so the `if` condition needs to check for `null`. The [`GetCurrentMethod()`](http://msdn.microsoft.com/en-us/library/system.reflection.methodbase.getcurrentmethod.aspx) method may throw a `TargetException` if the method was invoked with a late-binding mechanism, which is not what we're doing here.

The code above should be relatively safe and should prevent a stack overflow if an exception is thrown within the event handler.
