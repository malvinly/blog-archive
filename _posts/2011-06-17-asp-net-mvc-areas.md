---
layout: post
title: "ASP.NET MVC Areas"
date: 2011-06-17 00:47:55 +0000
categories: ["Programming"]
original_url: https://nivlam.wordpress.com/2011/06/16/asp-net-mvc-areas/
---

One of the new additions in ASP.NET MVC 2 was areas, which allows me to organize my projects into smaller sections. The ASP.NET MVC project that I've been working on has grown to a point where splitting it into smaller sections made sense. After adding areas, I noticed a few quirks.

![]({{ site.baseurl }}/assets/images/2011/06/newarea.png)

Here I've created a new area called "About" with a `HomeController`. I've also added a `HomeController` to my root controllers folder. At this point, I'd expect to be able to visit "/Home" and "/About/Home", each displaying their own pages. However, I received an error whenever I try to visit "/Home".

{% raw %}
```text
Multiple types were found that match the controller named 'Home'. This can happen if the route that services this request ('{controller}/{action}/{id}') does not specify namespaces to search for a controller that matches the request. If this is the case, register this route by calling an overload of the 'MapRoute' method that takes a 'namespaces' parameter.

The request for 'Home' has found the following matching controllers:
MvcDemo.Controllers.HomeController
MvcDemo.Areas.About.Controllers.HomeController
```
{% endraw %}

Turns out that I can't have duplicate controller names across my areas. To remedy this issue, I needed to make an adjustment to the default route found in `global.asax.cs`.

{% raw %}
```csharp
routes.MapRoute(
    "Default",
    "{controller}/{action}/{id}",
    new { controller = "Home", action = "Index", id = UrlParameter.Optional },
    new string[] { "MvcDemo.Controllers" }  
);
```
{% endraw %}

Adding a fourth argument to the `MapRoute` method informs the handler that it should look in the specified namespace first. If it can't find an appropriate controller in that namespace, fall back to the default behavior. After adding that, I was able to successfully visit "/Home" and "/About/Home" and hit the correct controllers.

![]({{ site.baseurl }}/assets/images/2011/06/newarea2.png)

Here I've created an additional controller in the About area named `ContactMeController`. I was able to successfully visit "/About/ContactMe" and have it display the correct page. When I visit "/ContactMe", I expected to receive a HTTP 404 back. However, I received a different error.

{% raw %}
```text
The view 'Index' or its master was not found or no view engine supports the searched locations. The following locations were searched:

~/Views/ContactMe/Index.aspx
~/Views/ContactMe/Index.ascx
~/Views/Shared/Index.aspx
~/Views/Shared/Index.ascx
~/Views/ContactMe/Index.cshtml
~/Views/ContactMe/Index.vbhtml
~/Views/Shared/Index.cshtml
~/Views/Shared/Index.vbhtml
```
{% endraw %}

This indicates that the handler found a controller, but the `ViewResult` could not find a view. This is the same scenario we encountered above with the two `HomeControllers` since the handler by default does not respect area namespaces. The namespace I added earlier was a namespace that has priority, but does not constrain it to that namespace. At this point, I want the controllers in the root controllers folder to be restricted to their namespace.

{% raw %}
```csharp
Route route = routes.MapRoute(
    "Default",
    "{controller}/{action}/{id}",
    new { controller = "Home", action = "Index", id = UrlParameter.Optional },
    new string[] { "MvcDemo.Controllers.*" }  
);

route.DataTokens["UseNamespaceFallback"] = false;
```
{% endraw %}

I've updated the namespace to include everything in that namespace and below. I've added an extra `DataToken` "UseNamespaceFallback" and set it to `false`. This extra `DataToken` indicates that the handler should not fall back to other namespaces in the project if it cannot find a controller in the specified namespace. Now I can visit "/ContactMe" and receive back the expected HTTP 404. At the same time, I can visit "/About/ContactMe" and have it return the correct page.
