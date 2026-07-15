---
layout: post
title: "Castle Windsor 3 - Classes and Types"
date: 2012-02-07 07:07:35 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2012/02/07/castle-windsor-3-classes-and-types/
---

In my previous post on [strongly typed sessions in ASP.NET](http://malvinly.com/2011/03/03/strongly-typed-asp-net-session/), I had to manually look through the current assembly and retrieve all interfaces that end in "Session". To register these interfaces with my container, I had to loop through each type found and register them one-by-one:

{% raw %}
```csharp
IEnumerable<Type> sessions =
    Assembly
        .GetExecutingAssembly()
        .GetTypes()
        .Where(x => x.IsInterface && x.Name.EndsWith("Session"));

foreach (Type session in sessions)
{
    Type sessionTemp = session;

    container.Register
    (
        Component
            .For(session)
            .UsingFactoryMethod(x =>
            {
                var factory = x.Resolve<DictionaryAdapterFactory>();
                return factory.GetAdapter<object>(sessionTemp, new SessionDictionary(HttpContext.Current.Session));
            })
            .LifeStyle.PerWebRequest
    );
```
{% endraw %}

Unfortunately, Castle Windsor 2.x did not have support for finding and registering interfaces with no implementions for use with a [factory method](http://stw.castleproject.org/Windsor.Registering-components-one-by-one.ashx#Using_a_delegate_as_component_factory_6) or [typed factories](http://stw.castleproject.org/Windsor.Typed-Factory-Facility-interface-based-factories.ashx). I also couldn't take advantage of the rich registration API of the `AllTypes` class since `AllTypes` only returned concrete classes.

Enter Castle Windsor 3. In version 3, Castle Windsor introduced two new registration classes, `Classes` and `Types`. The `Classes` class is identical to `AllTypes`. The new `Types` class returns all types, including interfaces, abstract classes, and concrete classes.

We can now update the registration code above to take advance of the new `Types` class:

{% raw %}
```csharp
container.Register
(
    Types
        .FromThisAssembly()
        .Where(t => t.IsInterface && t.Name.EndsWith("Session"))
        .Configure(c => c.UsingFactoryMethod(f =>
        {
            var factory = f.Resolve<DictionaryAdapterFactory>();
            return factory.GetAdapter<object>(c.Implementation, new SessionDictionary(HttpContext.Current.Session));
        })) 
        .LifestylePerWebRequest()
);
```
{% endraw %}
