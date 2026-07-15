---
layout: post
title: "Loading types at runtime"
date: 2011-09-23 11:43:31 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2011/09/23/loading-types-at-runtime/
---

I have a library that contains a factory, which creates instances of a type that implement an interface. Currently the factory, interface, and the implementations are in the same assembly. For maintainability, I'd like to separate the implementations from the core library and put them into their own assemblies. This will allow me to version each implementation independently from the core library. Any applications that need to use certain implementations will simply need to reference the core library and any number of libraries that contain the implementations. The factory in the core library should be able to determine which implementations are available at runtime.

Here I have a class library `Demo.Core` that contains an interface definition and a factory. There are no implementations for `IPrinter` in the core library. I have two additional class libraries called `Demo.GreenPrinter` and `Demo.RedPrinter` that contains implementations for `IPrinter`. The two libraries that contain the implementations references the core library, but the core library does not reference either of the libraries that contain the implementations. The console application needs to references the core library and at least one of the additional libraries.

![]({{ site.baseurl }}/assets/images/2011/09/solution_explorer.png)

The interface contains two methods. `CanHandle` contains logic to determine if a printer should be used and the `Print` method prints the message to the screen.

{% raw %}
```csharp
public interface IPrinter
{
    bool CanHandle(string color);
    void Print(string message); 
}
```
{% endraw %}

The managed extensibility framework (MEF) makes it easy for me to accomplish what I'm trying to do. The factory contains a static list of `IPrinter`s and is populated by the code in the static constructor using MEF. Unfortunately, MEF does not support static imports. I did not use MEF to automatically compose the `PrinterFactory` because I think it's unnecessary to new up instances of each printer when only one is going to be returned, hence the static list and constructor.

{% raw %}
```csharp
public class PrinterFactory
{
    private static IList<IPrinter> Printers { get; set; }

    static PrinterFactory()
    {
        var catalog = new DirectoryCatalog(Environment.CurrentDirectory, "Demo.*");
        var container = new CompositionContainer(catalog);

        Printers = container.GetExportedValues<IPrinter>().ToList();
    }
        
    public IPrinter Create(string color)
    {
        foreach (IPrinter printer in Printers)
        {
            if (printer.CanHandle(color))
                return printer;
        }

        return null;
    }
}
```
{% endraw %}

I'm using the `DirectoryCatalog` to find the assemblies. The core library and the libraries that contain the implementations will normally be copied to the same bin directory since they are both referenced by the console application. My console application can now create a `PrinterFactory` to create an `IPrinter`.

{% raw %}
```csharp
static void Main(string[] args)
{
	var factory = new PrinterFactory();
	RedPrinter printer = factory.Create("red") as RedPrinter;

	printer.Print("hello world");
}
```
{% endraw %}

This is all I would need to do if my core library was targeting .NET 4.0. However, if my core library needs to target an older version of the framework (2.0, 3.0, 3.5), I cannot use MEF without adding an extra dependency. At this point, I need to fall back to loading assemblies and using the `Activator` to create an instance of each `IPrinter`. The factory's static constructor now looks like:

{% raw %}
```csharp
static PrinterFactory()
{
	string[] paths = Directory.GetFiles(Environment.CurrentDirectory, "Demo.*.dll");

	foreach (string path in paths)
		Assembly.LoadFrom(path);

	Printers = AppDomain.CurrentDomain
		.GetAssemblies()
		.SelectMany(x => x.GetTypes())
		.Where(x => typeof(IPrinter).IsAssignableFrom(x) && !x.IsAbstract && !x.IsInterface)
		.Select(x => (IPrinter)Activator.CreateInstance(x))
		.ToList();
}
```
{% endraw %}
