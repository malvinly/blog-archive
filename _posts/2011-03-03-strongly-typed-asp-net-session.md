---
layout: post
title: "Strongly typed ASP.NET session"
date: 2011-03-03 17:47:26 +0000
categories: ["Programming"]
original_url: https://nivlam.wordpress.com/2011/03/03/strongly-typed-asp-net-session/
---

Following my previous post on [strongly typed dictionaries](http://malvinly.com/2011/02/28/strongly-typed-dictionaries/ "strongly typed dictionaries"), I wanted to use the same `DictionaryAdapter` component for ASP.NET sessions. However, the ASP.NET session object does not implement `IDictionary`. Since the session object acts like a dictionary, we'll need to wrap it. Using the session adapter found [here](http://blog.jason.smale.org/2010/08/dictionary-adapter-for-httpsessionstate_15.html), we can now utilize the `DictionaryAdapter` component to create strongly typed implementations.

{% raw %}
```csharp
DictionaryAdapterFactory factory = new DictionaryAdapterFactory();
IHomeSession session = factory.GetAdapter<IHomeSession>(new SessionDictionary(HttpContext.Current.Session));
```
{% endraw %}

Each of my controllers only use a subset of the information stored in session. To avoid creating a "master" interface that contains all the different keys, I can create multiple interfaces that only map values that the controller needs.

{% raw %}
```csharp
public interface IHomeSession
{
	string First { get; set; }
	string Last { get; set; }
	int? Age { get; set; }
}

public interface IAccountSession
{
	string First { get; set; }
	string Last { get; set; }
	string Middle { get; set; }
	int? Age { get; set; }
}

public interface IOtherSession
{
	[Key("OtherFirstName")]
	string First { get; set; }

	[Key("OtherLastName")]
	string Last { get; set; }
}
```
{% endraw %}

Each of these interfaces are injected into different controllers. The `Home` controller would be injected with `IHomeSession` and so forth. Since the `DictionaryAdapter` component maps the property name to keys, the `First` string property on `IHomeSession` would map to the same value as the `First` string property on `IAccountSession`. Although `IOtherSession` also has a `First` string property, I'm using the `Key` attribute to denote that it should map to the key `OtherFirstName`.

{% raw %}
```csharp
public class HomeController : Controller
{
	private readonly IHomeSession session;

	public HomeController(IHomeSession session)
	{
		this.session = session;
	}

	public ActionResult Index(string first, string last, int age)
	{
		this.session.First = first;
		this.session.Last = last;
		this.session.Age = age;

		return View();
	}
}
```
{% endraw %}

I can register each interface separately:

{% raw %}
```csharp
container.Register
(
    Component
        .For<IHomeSession>()
        .UsingFactoryMethod(x =>
        {
            DictionaryAdapterFactory factory = new DictionaryAdapterFactory();
            return factory.GetAdapter<IHomeSession>(new SessionDictionary(HttpContext.Current.Session));
        })
);
```
{% endraw %}

... but this quickly becomes a friction point if we have multiple interfaces and would like to create additional interfaces in the future. Instead, we can register the `DictionaryAdapterFactory` as a singleton and get a list of all the interfaces that match a certain convention. We'll loop through and register each one individually.

{% raw %}
```csharp
container.Register
    (
        Component
            .For<DictionaryAdapterFactory>()
            .LifeStyle.Singleton
    );

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
                        DictionaryAdapterFactory factory = x.Resolve<DictionaryAdapterFactory>();
                        return factory.GetAdapter<object>(sessionTemp, new SessionDictionary(HttpContext.Current.Session));
                    })
                .LifeStyle.PerWebRequest
        );
}
```
{% endraw %}

At this point, we can create additional interfaces that match the current convention and the container will automatically register and inject them.
