---
layout: post
title: "Postback and hiding fields"
date: 2012-05-01 06:35:48 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2012/05/01/postback-and-hiding-fields/
---

I recently stumbled across some ASP.NET code that looks similar to this:

{% raw %}
```csharp
protected void Page_Load(object sender, EventArgs e)
{
    if (!IsPostBack)
    {
        if (!HasPermissions())
            HideSensitiveFields();
    }
}
```
{% endraw %}

The code here checks if this is a normal page load and not a postback. If this is the initial page load, it'll check if the user may see sensitive (administrative) fields and hides them if the user does not have the appropriate permissions.

The problem is that it assumes all initial page loads will enter the `!IsPostBack` conditional block. Postback can be faked by passing in a few select query strings, such as \_\_EVENTTARGET.

This problem can be solved using event validation, encrypted viewstate, and other better coding practices, but that's a different subject and I won't dive into those in this post.

{% raw %}
```text
http://localhost:8080/Default.aspx?__eventtarget=
```
{% endraw %}

By default, ASP.NET doesn't check the HTTP verb, so we can pass a few select query string parameters and essentially spoof a postback. The URL above would bypass the `!IsPostBack` conditional block, allowing access to the sensitive fields.
