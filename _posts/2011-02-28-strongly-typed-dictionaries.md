---
layout: post
title: "Strongly typed dictionaries"
date: 2011-02-28 19:22:41 +0000
categories: ["Programming"]
tags: ["C#"]
original_url: https://nivlam.wordpress.com/2011/02/28/strongly-typed-dictionaries/
---

I recently stumbled across the [`DictionaryAdapter`](http://stw.castleproject.org/%28S%28nppam045y0sdncmbazr1ob55%29%29/Tools.Castle-DictionaryAdapter.ashx) component, which is part of the Castle Project. Some of the issues with storing settings in a centralized configuration file is that you’ll typically need to access these values using magic strings and receive back untyped data.

{% raw %}
```xml
<appSettings>
  <add key="MyFile" value="C:\temp\file.txt" />
  <add key="NumberOfItems" value="23" />
  <add key="SomeReallyLongAndUglyItemName" value="hello world" />
</appSettings>
```
{% endraw %}

The DictionaryAdapter solves these issues by creating a strongly typed wrapper around these key/value stores or dictionaries. We simply create an interface that matches what we expect to receive back:

{% raw %}
```csharp
public interface IApplicationConfiguration
{
    string MyFile { get; set; }
    int NumberOfItems { get; set; }

    [Key("SomeReallyLongAndUglyItemName")]
    string HelloText { get; set; }
}
```
{% endraw %}

`NumberOfItems` is now strongly typed and the property `HelloText` maps to the key `SomeReallyLongAndUglyItemName`.

{% raw %}
```csharp
DictionaryAdapterFactory factory = new DictionaryAdapterFactory();
var config = factory.GetAdapter<IApplicationConfiguration>(ConfigurationManager.AppSettings);

Console.WriteLine(config.MyFile);
Console.WriteLine(config.NumberOfItems);
Console.WriteLine(config.HelloText);
```
{% endraw %}

The default convention matches the name of the property of the interface to the key in the dictionary. There are various attributes that are available that allows you to map different keys to different properties.  For example, the `Key` attribute above allows you to map a property to an unrelated key in the dictionary. More information on attributes can be found on the [Castle Project wiki](http://stw.castleproject.org/%28S%28nppam045y0sdncmbazr1ob55%29%29/Tools.DictionaryAdapter-Customizing_Adapters.ashx).

Using the `DictionaryAdapter` to create strongly typed wrappers for configuration allows my tests that are dependent on these dictionaries to be more flexible.

{% raw %}
```csharp
var stub = MockRepository.GenerateStub<IApplicationConfiguration>();
stub.Stub(x => x.HelloText).Return("woot");
```
{% endraw %}
