---
layout: post
title: "\"DateTime.Now\" in unit tests"
date: 2012-05-09 07:08:39 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2012/05/09/datetime-now-in-unit-tests/
---

Inspired by [Ayende's post](http://ayende.com/blog/3408/dealing-with-time-in-tests) about dealing with time in tests, I started using his implementation of `SystemTime.Now()`.

{% raw %}
```csharp
public static class SystemTime
{
    public static Func<DateTime> Now = () => DateTime.Now;
}
```
{% endraw %}

We may use `SystemTime.Now()` anywhere in our code instead of `DateTime.Now`. This will allow us to control time in a deterministic fashion in order to keep our tests consistent. However, one of the downsides is that we have to remember to reset the time after each test.

{% raw %}
```csharp
[TestMethod]
public void DemoTest()
{
    SystemTime.Now = () => new DateTime(1970, 1, 1);
            
    // Test code...

    SystemTime.Now = () => DateTime.Now;
}
```
{% endraw %}

We can ease the pain by implementing this as part of the tear down process that runs after each test. I think that's an acceptable solution, but I still prefer all my test code to be as close together as possible. The tear down process may also run for tests that don't deal with `SystemTime.Now()`, so that's extra overhead for nothing.

Instead, I decided to add a few additions to `SystemTime`:

{% raw %}
```csharp
public class SystemTime : IDisposable
{
    private static Func<DateTime> now = () => DateTime.Now;

    public static DateTime Now
    {
        get { return now(); }
    }

    private SystemTime()
    {
    }

    public static SystemTime Context(DateTime dateTime)
    {
        now = () => dateTime;
        return new SystemTime();
    }

    public void Dispose()
    {
        now = () => DateTime.Now;
    }
}
```
{% endraw %}

I changed the `Now` property to type `DateTime` instead of `Func<DateTime>` so that it more closely resembles `DateTime.Now`. Implementing `IDisposable` gives us an easy way to reset `SystemTime.Now`.

{% raw %}
```csharp
[TestMethod]
public void DemoTest()
{
    using (SystemTime.Context(new DateTime(1970, 1, 1)))
    {
        // Test code...
    }
}
```
{% endraw %}

To demonstrate, we can run this code:

{% raw %}
```csharp
Console.WriteLine(SystemTime.Now);

using (SystemTime.Context(new DateTime(1970, 1, 1)))
{
    Console.WriteLine(SystemTime.Now);
}

Console.WriteLine(SystemTime.Now);
```
{% endraw %}

The output is:

{% raw %}
```text
5/9/2012 2:21:08 AM
1/1/1970 12:00:00 AM
5/9/2012 2:21:08 AM
```
{% endraw %}
