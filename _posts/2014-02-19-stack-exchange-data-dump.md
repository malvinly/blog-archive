---
layout: post
title: "Stack Exchange data dump"
date: 2014-02-19 05:18:47 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2014/02/18/stack-exchange-data-dump/
---

I was looking for an older [area 51 proposal](http://area51.stackexchange.com/) and saw it was closed due to inactivity. Fortunately for me, Stack Exchange provides a data dump of all the questions and answers in XML.

I searched for an existing program that would allow me to quickly import and view the questions and answers, but I didn't see anything that I could get running quickly. So instead, I just threw together a couple lines of code in LINQPad to view it. It's not clean and it'll probably throw an out of memory exception on larger files, but it works good enough for the data I have.

To view questions sorted by score:

{% raw %}
```csharp
XDocument
	.Load(@"C:\temp\posts.xml")
	.Element("posts")
	.Elements("row")
	.Where(x => x.Attribute("PostTypeId").Value == "1")
	.OrderByDescending(x => Int32.Parse(x.Attribute("Score").Value))
	.Select(x => 
		new 
		{
			Score = x.Attribute("Score").Value,
			Id = x.Attribute("Id").Value,
			Title = x.Attribute("Title").Value,
			Body = Util.RawHtml(WebUtility.HtmlDecode(x.Attribute("Body").Value)),
		})
	.Dump();
```
{% endraw %}

To view answers for a specific questions:

{% raw %}
```csharp
string parentId = "19422";

XDocument
	.Load(@"C:\temp\posts.xml")
	.Element("posts")
	.Elements("row")
	.Where(x => x.Attribute("ParentId") != null && x.Attribute("ParentId").Value == parentId)
	.OrderByDescending(x => Int32.Parse(x.Attribute("Score").Value))
	.Select(x => 
		new 
		{
			Score = x.Attribute("Score").Value,
			Body = Util.RawHtml(WebUtility.HtmlDecode(x.Attribute("Body").Value)),
		})
	.Dump();
```
{% endraw %}

To view a summary of all the questions and answers sorted by score:

{% raw %}
```csharp
void Main()
{
	var rows = XDocument
		.Load(@"C:\temp\posts.xml")
		.Element("posts")
		.Elements("row")
		.OrderBy(x => x.Attribute("PostTypeId").Value);		
		
	List<Question> threads = new List<Question>();
	
	foreach (var row in rows)
	{
		if (row.Attribute("PostTypeId").Value == "1")
		{
			var t = new Question
			{
				AcceptedAnswerId = row.Attribute("AcceptedAnswerId") != null ? row.Attribute("AcceptedAnswerId").Value : null,
				Answers = new List<Post>(),
				Id = row.Attribute("Id").Value,
				Body = row.Attribute("Body").Value,
				Title = row.Attribute("Title").Value,
				Score = Int32.Parse(row.Attribute("Score").Value)
			};
			
			threads.Add(t);
		}
		else if (row.Attribute("PostTypeId").Value == "2")
		{
			var parent = threads.FirstOrDefault(x => x.Id == row.Attribute("ParentId").Value);
					
			var t = new Post
			{
				Id = row.Attribute("Id").Value,
				Body = row.Attribute("Body").Value,
				Score = Int32.Parse(row.Attribute("Score").Value)
			};
			
			if (parent.AcceptedAnswerId == t.Id)
				parent.Answers.Insert(0, t);
			else
				parent.Answers.Add(t);
		}
	}
	
	threads.Sort((x, y) => y.Score.CompareTo(x.Score));
	
	foreach (var thread in threads)
		thread.Answers.Sort((x, y) => y.Score.CompareTo(x.Score));
		
	threads
		.Select(x => new 
		{
			Score = x.Score,
			Title = x.Title,
			Body = Util.RawHtml(WebUtility.HtmlDecode(x.Body)),
			Answers = x.Answers.Select(y => new
			{
				Score = y.Score,
				Body = Util.RawHtml(WebUtility.HtmlDecode(y.Body))
			})
		})
		.Dump();
}

public class Question : Post
{
	public string Title { get; set; }
	public string AcceptedAnswerId { get; set; }
	public List<Post> Answers { get; set; }
}

public class Post
{
	public string Id { get; set; }
	public int Score { get; set; }
	public string Body { get; set; }
}
```
{% endraw %}
