---
layout: post
title: "Logging database and web service calls with PostSharp"
date: 2015-11-04 22:42:36 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2015/11/04/logging-database-and-web-service-calls-with-postsharp/
---

We've been having some sporadic performance problems with our website. We needed to find out which page or operation was taking too long to complete and exhausting all the worker threads. Unfortunately, due to the size of the codebase and the lack of performance auditing, it was difficult to pinpoint where the problem was occurring.

Only a small portion of the codebase was utilizing dependency injection, so we couldn't decorate or intercept all dependencies with an auditing class. Fortunately, an IL weaving framework such as [PostSharp](https://www.postsharp.net/) can help with this situation. This was my first time using PostSharp, but I found it tremendously helpful.

A good would start would be logging database and web service calls.

{% raw %}
```csharp
using (SqlConnection conn = new SqlConnection("Server=.\\SQL2008; Database=Test; Trusted_Connection=True"))
using (SqlCommand cmd = new SqlCommand("cp_GetUserInfo", conn))
{
	cmd.CommandType = CommandType.StoredProcedure;
	cmd.Parameters.AddWithValue("@UserName", "JD");
	conn.Open();

	using (SqlDataReader reader = cmd.ExecuteReader())
	{
		while (reader.Read())
		{
			Console.WriteLine("User {0} {1}'s member ID is {2}.", reader["FirstName"], reader["LastName"], reader["MemberID"]);
		}
	}
}
```
{% endraw %}

Without modifying the existing code, I want to log the connection string, any parameters, and the amount of time it took to execute this stored procedure. We'll inherit from `OnMethodBoundaryAspect` to record information before and after our database call.

{% raw %}
```csharp
[AttributeUsage(AttributeTargets.Assembly)]
[MulticastAttributeUsage(MulticastTargets.Method)]
[Serializable]
public class DatabaseAspect : OnMethodBoundaryAspect
{
	public override void OnEntry(MethodExecutionArgs args)
	{
		args.MethodExecutionTag = Stopwatch.StartNew();
		
		SqlCommand cmd = (SqlCommand) args.Instance;

		Console.WriteLine("Executing command: {0}", cmd.CommandText);
		Console.WriteLine("\t- Connection String: {0}", cmd.Connection.ConnectionString);
		
		List<string> parameters = new List<string>();

		for (int i = 0; i < cmd.Parameters.Count; i++)
			parameters.Add("\t- Parameter " + cmd.Parameters[i].ParameterName + ": " + cmd.Parameters[i].Value);

		if (parameters.Count > 0)
			Console.WriteLine(String.Join("\n", parameters));
	}

	public override void OnExit(MethodExecutionArgs args)
	{
		Stopwatch sw = (Stopwatch) args.MethodExecutionTag;
		sw.Stop();

		SqlCommand cmd = (SqlCommand) args.Instance;

		Console.WriteLine("Command \"{0}\" took {1} ms.", cmd.CommandText, sw.ElapsedMilliseconds);
	}
}
```
{% endraw %}

Running the code will produce the following:

{% raw %}
```
Executing command: cp_GetUserInfo
        - Connection String: Server=.\SQL2008; Database=Test; Trusted_Connection=True
        - Parameter @UserName: JD
Command "cp_GetUserInfo" took 1124 ms.
User John Doe's member ID is 1234.
```
{% endraw %}

Great, now I can see the length of each database call and the parameters without digging through IIS logs hoping to find something to reconstruct the request. In additional to database calls, I want to log any web service requests. As with the database aspect, we can inherit from `OnMethodBoundaryAspect` to log information before and after the request.

{% raw %}
```csharp
[AttributeUsage(AttributeTargets.Assembly)]
[MulticastAttributeUsage(MulticastTargets.Method)]
[Serializable]
public class WebServiceAspect : OnMethodBoundaryAspect
{
	public override void OnEntry(MethodExecutionArgs args)
	{
		args.MethodExecutionTag = Stopwatch.StartNew();
		dynamic client = args.Instance;
		Console.WriteLine("Found service call to: {0}", client.Client.Endpoint.Address);
	}

	public override void OnExit(MethodExecutionArgs args)
	{
		Stopwatch sw = (Stopwatch) args.MethodExecutionTag;
		sw.Stop();

		dynamic client = args.Instance;
		Console.WriteLine("Service call to \"{0}\" took {1} ms.", client.Client.Endpoint.Address, sw.ElapsedMilliseconds);
	}
}
```
{% endraw %}

We'll apply these aspects to the entire assembly using assembly attributes.

{% raw %}
```csharp
[assembly: WebServiceAspect(AttributeTargetMembers = "regex:Return|Use", AttributeTargetAssemblies = "Example", AttributeTargetTypes = "Example.WcfClient*")]
[assembly: DatabaseAspect(AttributeTargetMembers = "Execute*", AttributeTargetAssemblies = "System.Data", AttributeTargetTypes = "System.Data.SqlClient.SqlCommand")]
```
{% endraw %}
