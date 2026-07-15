---
layout: post
title: "Archiving emails - Part II"
date: 2014-01-23 18:30:32 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2014/01/23/archiving-emails-part-ii/
---

After giving some more thought to my [last post](http://malvinly.com/2014/01/22/archiving-emails-the-hard-way/), I came up with a slightly better solution.

Due to the presence of the policy "PSTDisableGrow", Outlook cannot create new PST files or add mail to existing PST files. This basically means I can't use Outlook's archiving feature.

However, that doesn't stop me from creating an Outlook addin that can do exactly that. Instead of saving emails to my local machine as MSG files, I'll just move the emails into a new PST file that I create.

I can create a new PST file using the `NameSpace.AddStoreEx` method. If the PST file does not exist, Outlook will create it.

{% raw %}
```csharp
Application
	.Session
	.AddStoreEx(@"D:\EmailTest\ArchiveTest.pst", OlStoreType.olStoreDefault);
```
{% endraw %}

By default, the name of this new data file will be "Outlook Data File". There isn't any obvious way to change the display name, so I need to loop through all the root folders and look for my PST file. I can set the display name when I cast it to type `Folder`.

{% raw %}
```csharp
Folders folders = Application.Session.Folders;

for (int i = 1; i <= folders.Count; i++)
{
	Folder target = (Folder) folders[i];
	
	Store store = target.Store;
	string path = store.FilePath;
	Marshal.ReleaseComObject(store);
	
	if (path == @"D:\EmailTest\ArchiveTest.pst")
	{
		target.Name = "Archive Test";
		Marshal.ReleaseComObject(target);
		
		break;
	}
	
	Marshal.ReleaseComObject(target);
}

Marshal.ReleaseComObject(folders);
```
{% endraw %}

Now I can loop through my inbox, make a copy of the email, and save it to my new archive.

{% raw %}
```csharp
for (int i = 1; i <= inboxItems.Count; i++)
{
	var email = inboxItems[i] as MailItem;

	if (email == null)
		continue;

	MailItem copy = (MailItem) email.Copy();
	copy.Move(archiveInbox)

	Marshal.ReleaseComObject(copy);
	Marshal.ReleaseComObject(email);
}
```
{% endraw %}

In my previous post, I used the entry ID as the file name when I saved the email to my local machine. I used this ID as a unique identifier to determine which emails I have already saved. However, I cannot use the entry ID as a unique identifier when I move emails from my default mailbox to my archive. MAPI assigns a unique entry ID to each email that comes into a mailbox. However, that entry ID changes when it moves from one store (my default mailbox) to another store (my archive).

Instead, I can utilize user properties on the email itself. Each email contains a collection of user properties, which are just key value pairs. I can add my own custom user property to indicate that an email has already been archived.

{% raw %}
```csharp
for (int i = 1; i <= inboxItems.Count; i++)
{
	var email = inboxItems[i] as MailItem;

	if (email == null)
		continue;

	UserProperties userProperties = email.UserProperties;
	UserProperty archivedProperty = userProperties.Find("_archived");  

	if (archivedProperty == null)
	{      
		MailItem copy = (MailItem) email.Copy();
		copy.Move(archiveInbox)
		Marshal.ReleaseComObject(copy);
		
		userProperties.Add("_archived", OlUserPropertyType.olText, false, OlFormatText.olFormatTextText);
		email.Save();
	}
	else
		Marshal.ReleaseComObject(archivedProperty);

	Marshal.ReleaseComObject(userProperties);
	Marshal.ReleaseComObject(email);
}
```
{% endraw %}
