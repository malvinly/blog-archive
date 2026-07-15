---
layout: post
title: "Static fields and the \"BeforeFieldInit\" flag"
date: 2012-05-07 01:38:34 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2012/05/06/static-fields-and-the-beforefieldinit-flag/
---

While upgrading a legacy WinForm application from .NET 2.0 to .NET 4.0, I noticed some slightly different behavior. I'm working with a class that has a private static field and a few static methods.

{% raw %}
```csharp
public class Demo
{
	private static Expensive expensive = new Expensive();

	public static void Start() 
	{
	}

	public static void UseExpensive()
	{
		expensive.DoSomething();
	}
}

internal class Expensive
{
	public Expensive()
	{
		// Expensive operation
	}

	public void DoSomething()
	{
	}
}
```
{% endraw %}

The `Demo` class uses field initialization to create an instance of the `Expensive` class. The `Start` method does not use any of the static fields.

The WinForm application calls `Demo.Start()` in the constructor and `Demo.UseExpensive()` in a button event.

{% raw %}
```csharp
public partial class SomeForm : Form
{
	public SomeForm()
	{
		InitializeComponent();

		Demo.Start();
	}

	private void button_Click(object sender, EventArgs e)
	{
		Demo.UseExpensive();
	}
}
```
{% endraw %}

In .NET 2.0, the constructor of the `Expensive` class would execute on application start-up. In .NET 4.0 however, the constructor of the `Expensive` class would not execute until the button event was fired. Since the `Expensive` class is costly to create due to various operations in the constructor, the button event in .NET 4.0 was noticeably slower on the first click.

The reason is due to how .NET initializes types that contain static fields and/or a static constructor. If a type does not contain a static constructor, it will be marked with the `BeforeFieldInit` flag.

Using ILSpy to decompile into IL, we can see the current class marked with `BeforeFieldInit`:

[![]({{ site.baseurl }}/assets/images/2012/05/beforefieldinit.png "BeforeFieldInit")]({{ site.baseurl }}/assets/images/2012/05/beforefieldinit.png)

If we include a static constructor (even if it's empty), we'll see that the class is no longer marked with `BeforeFieldInit`:

[![]({{ site.baseurl }}/assets/images/2012/05/withoutbeforefieldinit.png "WithoutBeforeFieldInit")]({{ site.baseurl }}/assets/images/2012/05/withoutbeforefieldinit.png)

When marked with `BeforeFieldInit` in .NET 2.0, static field initialization may occur at any time before a static field is used. This means that static field initialization may occur even if a static field is never used. When marked with `BeforeFieldInit` in .NET 4.0, static field initialization only occurs before a static field is used. To demonstrate, we can use a console application to see when events occur.

{% raw %}
```csharp
class Program
{
	static void Main(string[] args)
	{
		Console.WriteLine("Started.");
		Demo.Start();
		Console.WriteLine("End.");
	}
}

public class Demo
{
	private static ExpensiveClass expensive = new ExpensiveClass();

	public static void Start()
	{
		Console.WriteLine("In Start method.");
	}

	public static void UseExpensive()
	{
		Console.WriteLine("In UseExpensive method.");
		expensive.DoSomething();
	}
}

internal class ExpensiveClass
{
	public ExpensiveClass()
	{
		Console.WriteLine("In ExpensiveClass constructor.");
	}

	public void DoSomething()
	{
	}
}
```
{% endraw %}

The console application is calling `Demo.Start()` in the `Main` method, which does not reference any static fields. The output in .NET 2.0 is:

{% raw %}
```text
In ExpensiveClass constructor.
Started.
In Start method.
End.
```
{% endraw %}

The output in .NET 4.0 is:

{% raw %}
```text
Started.
In Start method.
End.
```
{% endraw %}

Although the `Demo.Start()` method never references any static fields, static field initialization occurs anyway in .NET 2.0.

We can force static field initialization to occur in .NET 4.0 by adding a call to `Demo.UseExpensive()`, which does reference a static field.

{% raw %}
```csharp
static void Main(string[] args)
{
	Console.WriteLine("Started.");
	Demo.Start();
	Demo.UseExpensive();
	Console.WriteLine("End.");
}
```
{% endraw %}

The output in .NET 4.0 is:

{% raw %}
```csharp
Started.
In Start method.
In ExpensiveClass constructor.
In UseExpensive method.
End.
```
{% endraw %}

As you can see, static field initialization does not occur in .NET 4.0 until a static field is first referenced.

If we update the `Demo` class and add an empty static constructor, the class will no longer be marked with `BeforeFieldInit`. If a type isn't marked with `BeforeFieldInit`, static field initialization will occur before *any* class member is used.

{% raw %}
```csharp
class Program
{
	static void Main(string[] args)
	{
		Console.WriteLine("Started.");
		Demo.Start();
		Console.WriteLine("End.");
	}
}

public class Demo
{
	public static ExpensiveClass expensive = new ExpensiveClass();

	static Demo()
	{
	}

	public static void Start()
	{
		Console.WriteLine("In Start method.");
	}

	public static void UseExpensive()
	{
		Console.WriteLine("In UseExpensive method.");
		expensive.DoSomething();
	}
}
```
{% endraw %}

The output for .NET 2.0 and .NET 4.0 will be identical:

{% raw %}
```text
Started.
In ExpensiveClass constructor.
In Start method.
End.
```
{% endraw %}
