---
layout: post
title: "Check whether a .NET dll is built for Any CPU, x86, or x64"
date: 2016-11-16 20:56:27 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2016/11/16/check-whether-a-net-dll-is-built-for-any-cpu-x86-or-x64/
---

As much as I would like all builds to come from a build server, many times a build comes from a developer's machine. When I receive a build from another developer's machine, I need to check if their build is targeting the correct platform.

I can use CorFlags.exe, which is part of the .NET Framework SDK, to find out this information from a dll. Running CorFlags.exe with the file path to the dll will produce the following output:

{% raw %}
```
>> CorFlags "C:\example.dll"

Microsoft (R) .NET Framework CorFlags Conversion Tool.  Version  4.6.81.0
Copyright (c) Microsoft Corporation.  All rights reserved.

Version   : v4.0.30319
CLR Header: 2.5
PE        : PE32
CorFlags  : 0x3
ILONLY    : 1
32BITREQ  : 1
32BITPREF : 0
Signed    : 0
```
{% endraw %}

The two fields we need to look at are "PE" and "32BITREQ".

|  |  |
| --- | --- |
| Any CPU | PE: PE32, 32BITREQ: 0 |
| x86 | PE: PE32, 32BITREQ: 1 |
| x64 | PE: PE32+, 32BITREQ: 0 |

To programmatically determine the target platform, we can use [Module.GetPEKind()](https://msdn.microsoft.com/en-us/library/system.reflection.module.getpekind(v=vs.110).aspx).

{% raw %}
```csharp
Assembly a = Assembly.ReflectionOnlyLoadFrom(@"C:\example.dll");

PortableExecutableKinds peKind;
ImageFileMachine machine;

a.ManifestModule.GetPEKind(out peKind, out machine);

Console.WriteLine(peKind);
```
{% endraw %}

The results of peKind can be interpreted with:

|  |  |
| --- | --- |
| Any CPU | ILOnly |
| x86 | ILOnly, Required32Bit |
| x64 | ILOnly, PE32Plus |
