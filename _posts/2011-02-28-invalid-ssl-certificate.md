---
layout: post
title: "Invalid SSL certificate"
date: 2011-02-28 19:05:36 +0000
categories: ["Programming"]
tags: ["C#"]
original_url: https://nivlam.wordpress.com/2011/02/28/invalid-ssl-certificate/
---

While trying to invoke a web service in one of my development environments, I received this exception:

{% raw %}
```text
Could not establish trust relationship for the SSL/TLS secure channel.
```
{% endraw %}

There are a number of reasons why this may happen, but in my particular case I was using a self-signed SSL certificate for testing. To bypass the invalid certificate for testing, we can replace the certificate validation with our own:

{% raw %}
```csharp
ServicePointManager.ServerCertificateValidationCallback = delegate { return true; };
```
{% endraw %}

This implementation returns `true`, regardless if the certificate is valid or not.  One thing to keep in mind is that this setting will affect all subsequent requests from the current application domain.
