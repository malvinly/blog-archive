---
layout: post
title: "Extracting text messages on iPhone"
date: 2014-08-20 22:26:48 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2014/08/20/extracting-text-messages-on-iphone/
---

I bought an iPhone a few years ago and I'm about to switch phones. I didn't want to lose the history of my text messages, so I wanted a copy of all my text messages.

After manually backing up the phone using iTunes, we can navigate to where the backup is stored locally. In my case, the backup is stored at:

{% raw %}
```text
C:\Users\<MyUserName>\AppData\Roaming\Apple Computer\MobileSync\Backup\
```
{% endraw %}

All text messages are stored in a SQLite file named `3d0d7e5fb2ce288813306e4d4636395e047a3d28`. There are two tables I'm going to focus on: `message` and `handle`. The `message` table contains your entire history of text messages and various flags to determine whether a message was sent or received, whether the message was read, etc.... The `handle` table contains a list of phone numbers you've messaged and some additional information such as whether they were sent as a text message (SMS) or an iMessage.

Since I just wanted to backup my entire history, I can join these two tables using the following query:

{% raw %}
```sql
select 
	DATETIME(date + 978307200, "unixepoch", "localtime") as "Date", 
	h.id as "Phone Number", 
	m.text as "Text"
from message m inner join handle h on m.handle_id = h.rowid
order by m.rowid desc
```
{% endraw %}

I did have some trouble with the `date` column in the `message` table. The `date` column stores a number like `340475640`, which I wrongly assumed was the Epoch time that began on January 1, 1970. Turns out that dates in iOS starts on January 1, 2001. The magic number `978307200` in the query above represents how many seconds there are between these two dates.

Because LINQPad has support for SQLite using the IQ driver, it was easy to use LINQpad to query this database using both SQL and LINQ.

{% raw %}
```csharp
(from m in message
join h in handle
on m.handle_id equals h.ROWID
orderby m.date descending
select new 
{
	Date = new DateTime(2001, 1, 1, 0, 0, 0, 0, DateTimeKind.Utc)
		.AddSeconds(m.date.Value)
		.ToLocalTime(),
	PhoneNumber = h.id,
	Text = m.text
})
.Dump();
```
{% endraw %}
