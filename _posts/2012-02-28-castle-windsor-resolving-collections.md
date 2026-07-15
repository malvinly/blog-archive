---
layout: post
title: "Castle Windsor - resolving collections"
date: 2012-02-28 00:15:33 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2012/02/27/castle-windsor-resolving-collections/
---

Although I've used Castle Windsor on many projects, I often forget that Windsor does not automatically resolve dependencies that are a collection.

{% raw %}
```csharp
public class Demo
{
	private readonly IPrinter[] printers;

	public Demo(IPrinter[] printers)
	{
		this.printers = printers;
	}
}
```
{% endraw %}

Trying to call the `Resolve` method on the class above will throw an exception:

{% raw %}
```text
Castle.MicroKernel.Handlers.HandlerException: Can't create component 'CollectionResolverDemo.Demo' as it has dependencies to be satisfied.

'CollectionResolverDemo.Demo' is waiting for the following dependencies:
- Service 'CollectionResolverDemo.IPrinter[]' which was not registered.
```
{% endraw %}

Before registering components, we need to add a dependency resolver that can handle arrays, lists, and other collections:

{% raw %}
```csharp
container.Kernel.Resolver.AddSubResolver(new CollectionResolver(container.Kernel));
```
{% endraw %}

It is important to add the collection resolver before registering components. If the resolver is added after the component registration, collections will not be resolved correctly and the same exception will be thrown.

{% raw %}
```csharp
container.Register(Component.For<Demo>());
container.Register(Component.For<IPrinter>().ImplementedBy<RedPrinter>());
container.Register(Component.For<IPrinter>().ImplementedBy<GreenPrinter>());
container.Register(Component.For<IPrinter>().ImplementedBy<BluePrinter>());

// Bad
container.Kernel.Resolver.AddSubResolver(new CollectionResolver(container.Kernel));
```
{% endraw %}

One strange behavior that I've noticed is that if I add the collection resolver in the middle of my registration:

{% raw %}
```csharp
container.Register(Component.For<Demo>());
container.Register(Component.For<IPrinter>().ImplementedBy<RedPrinter>());
container.Register(Component.For<IPrinter>().ImplementedBy<GreenPrinter>());

container.Kernel.Resolver.AddSubResolver(new CollectionResolver(container.Kernel));

container.Register(Component.For<IPrinter>().ImplementedBy<BluePrinter>());
```
{% endraw %}

I would expect that BluePrinter would be the only printer that is passed to the `Demo` class. However, this is not the case. Instead, the entire collection of `IPrinter`s is resolved correctly.
