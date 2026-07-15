---
layout: post
title: "WCF clients and IDisposable"
date: 2012-05-01 07:19:04 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2012/05/01/wcf-clients-and-idisposable/
---

There are numerous articles detailing the "broken" IDisposable implementation for WCF clients. Just to name a few:

- <http://www.codeproject.com/Tips/197531/Do-not-use-using-for-WCF-Clients>
- <http://geekswithblogs.net/DavidBarrett/archive/2007/11/22/117058.aspx>
- <http://ardalis.com/idisposable-and-wcf>

To summarize, the `Close()` method will be called when the WCF client is disposed. However, if we try to dispose the client while it's in a faulted state, an exception will be thrown.

The extension method provided in the CodeProject article allows you do something like this:

{% raw %}
```csharp
new MyTestClient().Using(x => x.DoSomething());
```
{% endraw %}

The `Using()` extension method will automatically close or abort the client once finished. However, I do not like this extension method. Without any explanation, a developer may mistakenly reuse the client:

{% raw %}
```csharp
var client = new MyTestClient();
client.Using(x => x.DoSomething());
client.Using(x => x.DoSomething());  // Connection is already closed.
```
{% endraw %}

Instead, I use this class:

{% raw %}
```csharp
public static class Client
{
    public static void Use<TClient>(Action<TClient> action)
        where TClient : ICommunicationObject, new()
    {
        var client = new TClient();

        try
        {
            action(client);
            client.Close();
        }
        catch (Exception)
        {
            client.Abort();
            throw;
        }
    }
}
```
{% endraw %}

... and the usage:

{% raw %}
```csharp
Client.Use<MyTestClient>(x => x.DoSomething());
```
{% endraw %}

One problem with the class above is that if we needed to instantiate the client using one of the overloaded constructors, we couldn't due to the `new()` constraint. Instead, we can use this class:

{% raw %}
```csharp
public class Client<TClient>
    where TClient : ICommunicationObject
{
    private readonly TClient client;

    private Client(TClient client)
    {
        this.client = client;
    }

    public static Client<TClient> Instantiate(Func<TClient> action)
    {
        return new Client<TClient>(action());
    }

    public void Use(Action<TClient> action)
    {
        try
        {
            action(client);
            client.Close();
        }
        catch (Exception)
        {
            client.Abort();
            throw;
        }
    }
}
```
{% endraw %}

... and the usage:

{% raw %}
```csharp
Client<MyTestClient>
    .Instantiate(() => new MyTestClient("extra param"))
    .Use(x => x.DoSomething());
```
{% endraw %}
