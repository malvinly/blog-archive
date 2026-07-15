---
layout: post
title: "Multiple threads and refreshing Spring.NET's context"
date: 2012-10-04 07:39:04 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2012/10/04/multiple-threads-and-refreshing-spring-nets-context/
---

One of our legacy applications uses Spring.NET as a service locator. In most cases, I generally agree that using a [service locator is an anti-pattern](http://blog.ploeh.dk/2010/02/03/ServiceLocatorIsAnAntiPattern.aspx). Unfortunately, this code is littered throughout the project:

{% raw %}
```csharp
var context = ContextRegistry.GetContext() as IConfigurableApplicationContext;
IService service = context.GetObject("name-of-object") as IService;
```
{% endraw %}

Spring.NET exposes a method that refreshes all the object definitions.

{% raw %}
```csharp
var context = ContextRegistry.GetContext() as IConfigurableApplicationContext;
context.Refresh();
```
{% endraw %}

Unfortunately, if another thread is attempting to resolve an object from the container while the context is being refreshed, a `NullReferenceException` is thrown.

{% raw %}
```text
System.NullReferenceException: Object reference not set to an instance of an object.
   at Spring.Context.Support.AbstractApplicationContext.GetObject(String name) in c:\_prj\spring-net\trunk\src\Spring\Spring.Core\Context\Support\AbstractApplicationContext.cs:line 1538
   at ConsoleDemo.Program.<Main>b__1() in C:\projects\ConsoleDemo\ConsoleDemo\Program.cs:line 30
```
{% endraw %}

A simple solution is to utilize the [ReaderWriterLockSlim](http://msdn.microsoft.com/en-us/library/system.threading.readerwriterlockslim.aspx) class to allow multiple threads to resolve objects while allowing us to block all the threads when we need to refresh the object definitions.

{% raw %}
```csharp
public static class ServiceLocator
{
    private static readonly ReaderWriterLockSlim locker;

    static ServiceLocator()
    {
        locker = new ReaderWriterLockSlim();
    }

    public static T GetObject<T>(string name) 
        where T : class
    {
        locker.EnterReadLock();
        try
        {
            var context = ContextRegistry.GetContext() as IConfigurableApplicationContext;
            return context.GetObject(name) as T;
        }
        finally
        {
            locker.ExitReadLock();
        }
    }

    public static void RefreshContext()
    {
        locker.EnterWriteLock();
        try
        {
            var context = ContextRegistry.GetContext() as IConfigurableApplicationContext;
            context.Refresh();
        }
        finally
        {
            locker.ExitWriteLock();
        }
    }
}
```
{% endraw %}
