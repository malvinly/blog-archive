---
layout: post
title: "Executing code in a new application domain"
date: 2012-04-08 07:44:14 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2012/04/08/executing-code-in-a-new-application-domain/
---

A coworker recently mentioned a scenario where he needs to use legacy code in a multi-threaded environment. The class in-question is written in such a way that it holds state and needs to be reset before the class itself can be used again, therefore preventing it from being used in multiple threads at the same time. Although it may not be the most efficient method, we can create a new application domain for each thread and execute the code in there.

Just prototyping out some ideas, an API like this may be useful:

{% raw %}
```csharp
NewAppDomain.Execute(() =>
{
	// Run code in here
});
```
{% endraw %}

Each call to `Execute` would create a new application domain.

{% raw %}
```csharp
AppDomain domain = AppDomain.CreateDomain("New App Domain: " + Guid.NewGuid());
```
{% endraw %}

To execute our `Action` that we're passing to the `Execute` method, we'll need to create a new instance of `Action` in our new application domain. However, trying to call the `CreateInstanceAndUnwrap` method on the new application domain will throw an exception:

{% raw %}
```text
Unhandled Exception: System.MissingMethodException: Constructor on type 'System.Action' not found.
```
{% endraw %}

Instead, we'll need to create a new class that can be instantiated in the new application domain. This new class `AppDomainDelegate` will act as a delgate and execute the `Action` that we pass to it.

{% raw %}
```csharp
public static class NewAppDomain
{
    public static void Execute(Action action)
    {
        AppDomain domain = null;

        try
        {
            domain = AppDomain.CreateDomain("New App Domain: " + Guid.NewGuid());

            var domainDelegate = (AppDomainDelegate)domain.CreateInstanceAndUnwrap(
                typeof(AppDomainDelegate).Assembly.FullName,
                typeof(AppDomainDelegate).FullName);

            domainDelegate.Execute(action);
        }
        finally
        {
            if (domain != null)
                AppDomain.Unload(domain);
        }
    }

    private class AppDomainDelegate : MarshalByRefObject
    {
        public void Execute(Action action)
        {
            action();
        }
    }
}
```
{% endraw %}

Now we can use this class to execute code in another application domain.

{% raw %}
```csharp
NewAppDomain.Execute(() =>
{
    Console.WriteLine("Hello World");
});
```
{% endraw %}

**Parameters**

Trying to use variables declared outside the scope of the lambda will throw a serialization exception.

{% raw %}
```csharp
int i = 0;

NewAppDomain.Execute(() =>
{
	Console.WriteLine("Hello World " + i);
});
```
{% endraw %}

We can update the code to receive parameters across application domains by adding an overload for the `Execute` method:

{% raw %}
```csharp
public static class NewAppDomain
{
    public static void Execute(Action action) { ... }

    public static void Execute<T>(T parameter, Action<T> action)
    {
        AppDomain domain = null;

        try
        {
            domain = AppDomain.CreateDomain("New App Domain: " + Guid.NewGuid());

            var domainDelegate = (AppDomainDelegate)domain.CreateInstanceAndUnwrap(
                typeof(AppDomainDelegate).Assembly.FullName,
                typeof(AppDomainDelegate).FullName);

            domainDelegate.Execute(parameter, action);
        }
        finally
        {
            if (domain != null)
                AppDomain.Unload(domain);
        }
    }

    private class AppDomainDelegate : MarshalByRefObject
    {
        public void Execute(Action action) { ... }

        public void Execute<T>(T parameter, Action<T> action)
        {
            action(parameter);
        }
    }
}
```
{% endraw %}

Now we can pass parameters to the method. We'll need to make sure that any parameters we pass are marked as `Serializable` since they will be serialized and deserialized across application domains.

{% raw %}
```csharp
int i = 0;

NewAppDomain.Execute(i, x =>
{
    Console.WriteLine("Hello World " + x);
});
```
{% endraw %}

**Returning Results**

In cases where we want the `Execute` method to return results, we can add two additional method overloads that use `Func<T>` instead of `Action`:

{% raw %}
```csharp
public static T Execute<T>(Func<T> action)
{
    AppDomain domain = null;

    try
    {
        domain = AppDomain.CreateDomain("New App Domain: " + Guid.NewGuid());

        var domainDelegate = (AppDomainDelegate)domain.CreateInstanceAndUnwrap(
            typeof(AppDomainDelegate).Assembly.FullName,
            typeof(AppDomainDelegate).FullName);

        return domainDelegate.Execute(action);
    }
    finally
    {
        if (domain != null)
            AppDomain.Unload(domain);
    }
}

public static TResult Execute<T, TResult>(T parameter, Func<T, TResult> action)
{
    AppDomain domain = null;

    try
    {
        domain = AppDomain.CreateDomain("New App Domain: " + Guid.NewGuid());

        var domainDelegate = (AppDomainDelegate)domain.CreateInstanceAndUnwrap(
            typeof(AppDomainDelegate).Assembly.FullName,
            typeof(AppDomainDelegate).FullName);

        return domainDelegate.Execute(parameter, action);
    }
    finally
    {
        if (domain != null)
            AppDomain.Unload(domain);
    }
}
```
{% endraw %}

... and the same overloads for the delegate class `AppDomainDelegate`:

{% raw %}
```csharp
public T Execute<T>(Func<T> action)
{
    return action();
}

public TResult Execute<T, TResult>(T parameter, Func<T, TResult> action)
{
    return action(parameter);
}
```
{% endraw %}

We can now get results back. We'll need to make sure that any results we return are marked as `Serializable`.

{% raw %}
```csharp
Result first = NewAppDomain.Execute(() => new Result());
Result second = NewAppDomain.Execute(parameter, x => new Result());
```
{% endraw %}

**Using The Class**

The entire class can be found [here](https://gist.github.com/2335382). To demonstrate, we'll create a class that uses a `public static` field as a counter.

{% raw %}
```csharp
public class SharedClass : MarshalByRefObject
{
    public static int Counter = 1;

    public void Print()
    {
        Console.WriteLine(AppDomain.CurrentDomain.FriendlyName + " - " + Counter);
        Counter++;
    }
}
```
{% endraw %}

We'll call the `Print` method in another application domain using our `NewAppDomain` class:

{% raw %}
```csharp
NewAppDomain.Execute(() =>
{
	new SharedClass().Print();
	new SharedClass().Print();
});

NewAppDomain.Execute(() =>
{
	new SharedClass().Print();
	new SharedClass().Print();
	new SharedClass().Print();
	new SharedClass().Print();
});
```
{% endraw %}

The result:

{% raw %}
```text
New App Domain: 3ca2e6de-a5bd-40ef-b471-edb8c19024c2 - 1
New App Domain: 3ca2e6de-a5bd-40ef-b471-edb8c19024c2 - 2
New App Domain: 2037d461-79fe-48c1-abf1-2c58bb6b02cf - 1
New App Domain: 2037d461-79fe-48c1-abf1-2c58bb6b02cf - 2
New App Domain: 2037d461-79fe-48c1-abf1-2c58bb6b02cf - 3
New App Domain: 2037d461-79fe-48c1-abf1-2c58bb6b02cf - 4
```
{% endraw %}

If the application domain was the same, we would have seen the counter increment to 6. Notice that `SharedClass` inherits from `MarshalByRefObject`. If we instantiate a new instance of `SharedClass` and pass it as a parameter to our `Execute` method, this object is now shared across application domains via remoting. To demonstrate, we can use the `Execute` overload that takes a parameter:

{% raw %}
```csharp
var sharedClass = new SharedClass();

sharedClass.Print();
sharedClass.Print();

NewAppDomain.Execute(sharedClass, x => x.Print());

sharedClass.Print();
```
{% endraw %}

The result:

{% raw %}
```text
AppDomainDemo.exe - 1
AppDomainDemo.exe - 2
AppDomainDemo.exe - 3
AppDomainDemo.exe - 4
```
{% endraw %}

As expected, the counter did not reset. However, if we change `SharedClass` by removing `MarshalByRefObject` and marking the class as `Serializable`, we'll see a different result:

{% raw %}
```text
AppDomainDemo.exe - 1
AppDomainDemo.exe - 2
New App Domain: b9404ee9-f4fc-4937-84ad-9d0369e36bb6 - 1
AppDomainDemo.exe - 3
```
{% endraw %}
