---
layout: post
title: "Creating new application domains while running a unit test"
date: 2013-07-30 21:33:28 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2013/07/30/creating-new-application-domains-while-running-a-unit-test/
---

While using MSTest to test some code that creates new application domains, I kept running into an exception while using both Visual Studio's test runner and ReSharper's test runner. This code works under normal circumstances, but fails to run during the execution of a unit test.

{% raw %}
```csharp
[TestMethod]
public void Test1()
{
    AppDomain domain = AppDomain.CreateDomain("Test");
    domain.DoCallBack(() => Console.WriteLine("Hello world"));
}
```
{% endraw %}

Line 5 would throw the following exception:

{% raw %}
```text
Test method TestProject.DemoTest.Test1 threw exception: 
System.IO.FileNotFoundException: Could not load file or assembly 'TestProject, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null' or one of its dependencies. The system cannot find the file specified.
```
{% endraw %}

After some trial and error, I discovered the newly created application domain uses a different application base.

{% raw %}
```csharp
AppDomain domain = AppDomain.CreateDomain("Test");

Console.WriteLine(AppDomain.CurrentDomain.SetupInformation.ApplicationBase);
Console.WriteLine(domain.SetupInformation.ApplicationBase);
```
{% endraw %}

Running the code above during the execution of a normal application (ie console application) will print out the same location. In my case, it would be `"D:\projects\DemoCode\bin\Debug"`. But when this code is executed in the context of a test runner, the newly created application domain will print `"C:\Program Files (x86)\JetBrains\ReSharper\v6.1\Bin\"` while using ReSharper's test runner or `"C:\Program Files (x86)\Microsoft Visual Studio 10.0\Common7\IDE\"` while using Visual Studio's test runner.

To make sure both application domains use the same application base, I had to modify how I created new application domains.

{% raw %}
```csharp
AppDomain domain = AppDomain.CreateDomain(
	"Test",
	new Evidence(AppDomain.CurrentDomain.Evidence),
	new AppDomainSetup
	{
		ApplicationBase = AppDomain.CurrentDomain.SetupInformation.ApplicationBase,
	});
```
{% endraw %}
