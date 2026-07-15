---
layout: post
title: "MITM using Chromium Embedded Framework and Fiddler"
date: 2016-10-04 07:24:31 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2016/10/04/mitm-using-chromium-embedded-framework-and-fiddler/
---

For some background, I use a popular website that shows certain artifacts around my city. When a location is clicked, it'll show what artifacts are available at that location. The website works by sending an AJAX request to an API when a location is clicked and returns a JSON serialized list of artifacts.

A few weeks ago, I wrote a script to call their API directly. The script contained an infinite loop to call their API every minute and notify me when it found new artifacts. I could run this script when I was sleeping or away from my computer and it would text me the details using Twilio when it found something.

However, the website started cracking down on unsolicited API calls to their service a few days ago. Their first attempt was to generate a unique token each time a user visits their website and that token was included as part of the API call. This was simple to bypass since my script could simply scrap the website for the token.

They wised up pretty quickly. A few days later, they started obfuscating how the token was generated and each API call required a new token. I spent an hour trying to deobfuscate how they generated the token, but I couldn't figure it out without spending a large amount of time on it.

Fortunately, there's still a way around it. Chromium Embedded Framework (CEF) allows me to embed a headless browser into my script that is able to execute JavaScript and everything else a normal user would be able to do.

I can load the website using CEF and inject JavaScript into the website to click on links.

{% raw %}
```csharp
var browser = new ChromiumWebBrowser("https://example.com");

while (!browser.IsBrowserInitialized)
{
	Console.WriteLine("Waiting for browser initialization...");
	Thread.Sleep(TimeSpan.FromSeconds(2));
}

while (browser.IsLoading)
{
	Console.WriteLine("Waiting for website to load...");
	Thread.Sleep(TimeSpan.FromSeconds(2));
}

Console.WriteLine("Finished browser initialization.");
			
await browser.EvaluateScriptAsync("$('#location').trigger('click'););
```
{% endraw %}

Triggering a click event will cause the website to generate a new token and submit an AJAX request to their API. Unfortunately, I couldn't find any CEF documentation that showed me how to intercept the AJAX response.

Instead of trying to intercept the AJAX response using CEF, I used FiddlerCore to capture the AJAX response. FiddleCore can see all the traffic between CEF and their API.

{% raw %}
```csharp
FiddlerApplication.Startup(443, FiddlerCoreStartupFlags.Default);

FiddlerApplication.BeforeResponse += session =>
{
	if (session.RequestMethod == "CONNECT" || !session.LocalProcess.StartsWith("myApplication"))
		return;
	
	if (!session.HostnameIs("api.example.com"))
		return;
		
	dynamic result = JObject.Parse(session.GetResponseBodyAsString());
	
	// Do something with result
}
```
{% endraw %}

Instead of calling their API directly, I had to jump through a few hoops to get the data I wanted. Injecting click events into the website through CEF allowed me to automate the API calls without having to decipher the token generation. FiddleCore allowed me to monitor the traffic between CEF and their API.
