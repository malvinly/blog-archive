---
layout: post
title: "ASP.NET MVC CDN UrlHelper"
date: 2012-05-07 03:04:19 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2012/05/06/asp-net-mvc-cdn-urlhelper/
---

We're starting to move static files off our production web server to a content delivery network (CDN). Most of our existing views link to static resources using the `Url.Content()` helper.

{% raw %}
```html
<link href="@Url.Content("~/Content/Site.css")" rel="stylesheet" type="text/css" />
<script type="text/javascript" src="@Url.Content("~/Scripts/jquery.js")"></script>
```
{% endraw %}

While in development, I would like the resources to point to the local versions. In production, the resources should point to our CDN. I also want the ability to switch to the local versions if there is a problem with our CDN.

I've added two AppSettings keys to the web.config:

{% raw %}
```xml
<add key="CDN.Enabled" value="true" />
<add key="CDN.Address" value="http://cdn.example.com/" />
```
{% endraw %}

If we need to switch to local versions, we can flip `CDN.Enabled` to `false`. Instead of using the default `Url.Content()` helper, we'll update the views to use our own `Url.ContentFromCdn()` helper.

{% raw %}
```html
<link href="@Url.ContentFromCdn("~/Content/Site.css")" rel="stylesheet" type="text/css" />
<script type="text/javascript" src="@Url.ContentFromCdn("~/Scripts/jquery.js")"></script>
```
{% endraw %}

Here is the code for the helper:

{% raw %}
```csharp
public static class UrlHelpers
{
    public static string ContentFromCdn(this UrlHelper helper, string contentPath)
    {
        bool cdnEnabled;

        Boolean.TryParse(
            ConfigurationManager.AppSettings["CDN.Enabled"],
            out cdnEnabled);

        if (!cdnEnabled)
            return helper.Content(contentPath);
            
        var baseUri = new Uri(ConfigurationManager.AppSettings["CDN.Address"]);
        var content = VirtualPathUtility.ToAbsolute(contentPath);

        return new Uri(baseUri, content).AbsoluteUri;
    }
}
```
{% endraw %}

The helper will fall back to the default `Url.Content()` helper if the `CDN.Enabled` setting evaluates to `false`. If we need to use Google's CDN (or any other public CDN), we can use a dictionary to look for specific filenames and return a specific address.

{% raw %}
```csharp
public static class UrlHelpers
{
    private static Dictionary<string, string> cdn = 
        new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase)
        {
            { "jquery.js", "https://ajax.googleapis.com/ajax/libs/jquery/1.7.2/jquery.js" },
            { "jquery-ui.js", "https://ajax.googleapis.com/ajax/libs/jqueryui/1.8.18/jquery-ui.js" }
        };

    public static string ContentFromCdn(this UrlHelper helper, string contentPath)
    {
        bool cdnEnabled;

        Boolean.TryParse(
            ConfigurationManager.AppSettings["CDN.Enabled"],
            out cdnEnabled);

        // Use default Url.Content() helper.
        if (!cdnEnabled)
            return helper.Content(contentPath);

        var fileName = VirtualPathUtility.GetFileName(contentPath);

        // Use public CDN address for this file.
        if (cdn.ContainsKey(fileName))
            return cdn[fileName];
            
        var baseUri = new Uri(ConfigurationManager.AppSettings["CDN.Address"]);
        var content = VirtualPathUtility.ToAbsolute(contentPath);

        return new Uri(baseUri, content).AbsoluteUri;
    }
}
```
{% endraw %}
