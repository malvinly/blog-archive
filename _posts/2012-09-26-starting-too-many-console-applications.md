---
layout: post
title: "Starting too many console applications"
date: 2012-09-26 06:25:08 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2012/09/26/starting-too-many-console-applications/
---

We have a service that starts multiple instances of a console application using `Process.Start()`. During peak hours, there could potentially be 100 or more instances of the console application running. Obviously this isn't an ideal solution, but it's a legacy system that I'm currently supporting (if my boss is reading this, we need to redesign this system).

After a while, I noticed that many of the console applications weren't starting and there were no indications of any problems until I looked at the event logs.

{% raw %}
```text
ConsoleApplication.exe - Application Error : The application was unable to start correctly (0xc0000142). Click OK to close the application. 
```
{% endraw %}

Searching online for the error `0xc0000142` didn't yield much information. It seemed to be a generic error message that means either you're out of memory or the application is corrupt. This problem took me a few weeks to diagnose. At first, I just shrugged it off as an isolated incident without looking for an exact reason. Unfortunately, this happened again recently which prompted me to look for the root cause.

The service that's starting these console applications used to be load balanced over two servers. We recently moved this service onto just a single server since CPU and memory usage were fairly low. What I found out was that starting all these console applications resulted in desktop heap exhaustion.

This is a great article describing what the desktop heap is:

- [Desktop Heap Overview Part 1](http://blogs.msdn.com/b/ntdebugging/archive/2007/01/04/desktop-heap-overview.aspx)
- [Desktop Heap Overview Part 2](http://blogs.msdn.com/b/ntdebugging/archive/2007/07/05/desktop-heap-part-2.aspx)

> Every desktop object has a single desktop heap associated with it. The desktop heap stores certain user interface objects, such as windows, menus, and hooks. When an application requires a user interface object, functions within user32.dll are called to allocate those objects.

Part 2 of the article mentions that the non-interactive heap size for services is 512 KB. After some testing, I was able to confirm that the `0xc0000142` error started happening when we reached approximately ~120 console applications. The [desktop heap monitor](http://www.microsoft.com/en-us/download/details.aspx?id=17782) confirmed that heap utilization was high.

At this point, I would have to assume that we never exhausted the desktop heap when our services were load balanced across two servers. Since migrating to a single server, our desktop heap size was reduced by half.
