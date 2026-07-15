---
layout: post
title: "Archiving emails... the hard way"
date: 2014-01-23 02:30:03 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2014/01/22/archiving-emails-the-hard-way/
---

Email storage can be a problem. Many email providers limit the storage size for a given user. The wrong way to handle email storage is to limit how long an email can be kept.

Unfortunately, this is something I have to deal with. For whatever reason, the powers that be decided people don't need emails longer then three months, so emails older then three months will be automatically deleted. I find the whole situation comical, but that's an entirely different conversation.

Since I use Outlook/Exchange for these emails, a normal person would recommend archiving my emails to a PST file on my local machine. Unfortunately, a group policy was pushed out that added the registry key "PSTDisableGrow" for Outlook. This prevents Outlook from adding emails to PST files, even if it's stored locally.

So now I'm stuck in a position where I can't automatically archive my emails without paying for a third party product. I need a way to automatically save all my emails as either MSG or EML files to my hard drive, so at least I have a copy.

There are a couple options that I'm exploring. Be warned that the solutions I'm going to talk about are TERRIBLE. They are very much hacks and something that I would completely avoid if I had a chance. I'm open to any suggestions and/or free products.

The first thing I tried was to create a new rule in Outlook for all incoming emails. This rule would execute a custom VB script that saves a copy of the email to my machine. Unfortunately, I couldn't get it to work. I fumbled around with it for about an hour before I gave up.

The second option was to utilize [Exchange Web Services (EWS)](http://msdn.microsoft.com/en-us/library/dd877045(v=exchg.140).aspx). Newer versions of Exchange expose a SOAP web service for anyone to use. Most of the time, the location of the web service can be discovered by going to the address <http://webmail.example.com/ews/exchange.asmx>, where "example.com" is your domain. Microsoft provides a managed interface called [Exchange Web Services Managed API](http://msdn.microsoft.com/en-us/library/office/dd633709(v=exchg.80).aspx) that simplifies access. I was quite surprised at how easy it was to develop a simple solution.

{% raw %}
```csharp
var service = new ExchangeService(ExchangeVersion.Exchange2010_SP1)
{
	Credentials = new WebCredentials("user", "password"),
	Url = new Uri("https://webmail.example.com/ews/exchange.asmx")
};

Folder folder = Folder.Bind(service, WellKnownFolderName.Inbox);
FindItemsResults<Item> emails = folder.FindItems(new ItemView(Int32.MaxValue));

service.LoadPropertiesForItems(emails, new PropertySet(ItemSchema.MimeContent));

string archiveDirectory = Path.Combine(@"D:\EmailArchive", DateTime.Now.ToString("yyyy-MM"));

if (!Directory.Exists(archiveDirectory))
	Directory.CreateDirectory(archiveDirectory);

foreach (Item email in emails)
{
	string path = Path.Combine(archiveDirectory, email.StoreEntryId + ".eml");

	if (!File.Exists(path))
		File.WriteAllBytes(path, email.MimeContent.Content);
}
```
{% endraw %}

This code snippet basically downloads my entire inbox and saves it locally. It doesn't get any easier than that. EWS also supports streaming, push, and pull notifications. This allows me to monitor any incoming/outgoing emails and immediately archive them. I could fall back to iterating over the entire inbox every few days to catch any emails I missed.

As much as I like how simple this solution is, I can't depend on it. Unfortunately, EWS can be disabled by an Exchange administrator. Knowing how people have reacted before, this feature of Exchange will probably be disabled once they realize someone is using it.

The final option I'm currently exploring is to create an Outlook addin using VSTO. Unfortunately, it utilizes COM objects for nearly everything. I have very little experience with COM, so I ran into several issues.

{% raw %}
```csharp
Folder inbox = (Folder) Application.Session.GetDefaultFolder(OlDefaultFolders.olFolderInbox);
string archiveDirectory = Path.Combine(@"D:\EmailArchive", DateTime.Now.ToString("yyyy-MM"));

if (!Directory.Exists(archiveDirectory))
	Directory.CreateDirectory(archiveDirectory);

foreach (object item in inbox.Items)
{
	var email = item as MailItem;

	if (email == null)
		continue;

	string path = Path.Combine(archiveDirectory, email.EntryID + ".msg");

	if (!File.Exists(path))
		email.SaveAs(path);
}
```
{% endraw %}

There are several things wrong with the code above. Since nearly everything is a COM object, I need to release them after I'm done. The first time I ran this, it worked for the first few hundred emails. At the 300 mark, I received the exception "Your server administrator has limited the number of items you can open simultaneously." In the code above, each iteration of the loop would reference a new COM object. After a couple hundred iterations of the loop, it would fail because I never released any of them.

[This article](http://www.add-in-express.com/creating-addins-blog/2013/11/05/release-excel-com-objects/) mentions a pretty good guideline.

> 1 dot good, 2 dots bad

This means I need to pay special attention to property chaining. For example:

{% raw %}
```csharp
Folder inbox = (Folder) Application.Session.GetDefaultFolder(OlDefaultFolders.olFolderInbox);

// Bad
inbox.Items.ItemAdd += OnItemAdd;

// Good
Items inboxItems = inbox.Items;
inboxItems.ItemAdd += OnItemAdd;
```
{% endraw %}

Since I need to release the COM objects in the opposite order of creation, I used a stack to keep track of all my references.

{% raw %}
```csharp
Stack<object> comObjects = new Stack<object>();

Folder inbox = (Folder) Application.Session.GetDefaultFolder(OlDefaultFolders.olFolderInbox);
comObjects.Push(inbox);

Items inboxItems = inbox.Items;
comObjects.Push(inboxItems);

Folder sent = (Folder) Application.Session.GetDefaultFolder(OlDefaultFolders.olFolderSentMail);
comObjects.Push(sent);

Items sentItems = sent.Items;
comObjects.Push(sentItems);

// 
// Do something
//

while (comObjects.Count != 0)
{
	object obj = comObjects.Pop();

	if (obj != null)
		Marshal.ReleaseComObject(obj);
}
```
{% endraw %}

While iterating through the `Items` collection using a `for` loop, I would immediately receive an exception saying that the array was out-of-bounds. Like any C# developer, I start the iterating arrays at index 0. However, the `Items` collection starts at index 1. MSDN documents it [here](http://msdn.microsoft.com/en-us/library/office/ff864247.aspx).

The items collection contains an event that fires for each new email that is received. I can utilize this event for both the inbox and sent folders to immediately archive new emails. There is caveat to using this event that is mentioned in [this article](http://www.add-in-express.com/creating-addins-blog/2008/04/25/newmail-itemadd-outlook-events/). Whenever 16 or more items are added at the same time, this event does not fire. I don't need to worry about this limitation most of the time, but I would still need to fall back to iterating over the entire inbox every once in a while to make sure all the emails have been saved.

Creating an Outlook addin isn't as simple as using the EWS Managed API. With the restrictions in place, it seems this is the only option I currently have. Even after I save all the emails to my local machine, I still need to create a separate service to parse and index all the emails.

A few people have started migrating all their old emails into OneNote by highlighting their entire inbox and pressing one button. If things get too complicated, I might have to fall back to this.

This is a lot of work for a simple problem. The wrong way to handle email storage is to limit how long an email can be kept.
