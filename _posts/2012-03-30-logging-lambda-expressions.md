---
layout: post
title: "Logging lambda expressions"
date: 2012-03-30 05:42:30 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2012/03/30/logging-lambda-expressions/
---

When working with lambda expressions, there are scenarios where I need to log when a `Func<bool>` returns false. In the example below, just logging "condition was not met" doesn't give me very much information.

{% raw %}
```csharp
public void SomeMethod(Func<Person, bool> condition)
{
	if (!condition(person))
	{
		Log.Info("Condition was not met.");
		return;
	}

	// Do something
}
```
{% endraw %}

If I called the method like so:

{% raw %}
```csharp
obj.SomeMethod(x => x.Name == "Malvin");
```
{% endraw %}

... I would like to be able to turn that lambda expression into a meaningful string.

Unfortunately, there doesn't appear to be an easy way to turn a lambda expression into a readable string. Instead, I had to use expression trees.

{% raw %}
```csharp
public void SomeMethod(Expression<Func<Person, bool>> expression)
{
	if (!expression.Compile()(person))
	{
		Log.Info("Condition was not met: " + expression);
		return;
	}

	// Do something
}
```
{% endraw %}

Now if I call the method and the condition fails, I should see:

{% raw %}
```text
Condition was not met: x => (x.Name == "Malvin")
```
{% endraw %}

However, expression trees are much more expensive than lambda expressions. Running a non-scientific benchmark on my Core i7 @ 3.2 GHz yields some fairly obvious results.

{% raw %}
```csharp
void Main()
{
	Stopwatch sw = Stopwatch.StartNew();
	
	for (int i = 0; i < 100000; i++)
		UsingExpression(() => false);
	
	sw.Stop();
	
	Console.WriteLine ("Expression: " + sw.ElapsedMilliseconds);
	
	sw.Restart();
	
	for (int i = 0; i < 100000; i++)
		UsingLambda(() => false);
		
	sw.Stop();
	
	Console.WriteLine ("Lambda: " + sw.ElapsedMilliseconds);
}

public void UsingExpression(Expression<Func<bool>> expression)
{
	if (expression.Compile()())
		Console.WriteLine ("hello world");
}

public void UsingLambda(Func<bool> condition)
{
	if (condition())
		Console.WriteLine ("hello world");
}
```
{% endraw %}

Output:

{% raw %}
```text
Expression: 4154
Lambda: 0
```
{% endraw %}

This may be a problem for high volume systems, but I don't think the extra overhead will be an issue for most cases.
