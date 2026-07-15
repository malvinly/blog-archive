---
layout: post
title: "Using Web Forms user controls in an ASP.NET MVC project"
date: 2011-02-28 19:45:19 +0000
categories: ["Programming"]
tags: ["ASP.NET", "C#"]
original_url: https://nivlam.wordpress.com/2011/02/28/using-web-forms-user-controls-in-an-asp-net-mvc-project/
---

We have an existing ASP.NET Web Forms project that we’re in the process of slowly converting to ASP.NET MVC. We do not have the manpower to simply rewrite the entire website from scratch, so one of our goals is to slowly convert the website piece by piece. This means that the existing Web Forms project will need to coexist with emerging MVC project.

In the current Web Forms project, we have a number of user controls that are used on all our pages. In order to maintain backwards compatibility with our Web Forms pages, we couldn’t rewrite the controls as MVC helpers without having to duplicate code.

I created this helper that allows us to reuse Web Forms user controls on views:

{% raw %}
```csharp
public static class UserControlHelper
{
    public static HtmlString RenderControl<T>(this HtmlHelper helper, string path)
        where T : UserControl
    {
        return RenderControl<T>(helper, path, null);
    }
 
    public static HtmlString RenderControl<T>(this HtmlHelper helper, string path, Action<T> action)
        where T : UserControl
    {
        Page page = new Page();
        T control = (T)page.LoadControl(path);
        page.Controls.Add(control);
 
        if (action != null)
            action(control);
 
        using (StringWriter sw = new StringWriter())
        {
            HttpContext.Current.Server.Execute(page, sw, false);               
            return new HtmlString(sw.ToString());
        }
    }
}
```
{% endraw %}

By instantiating a `Page` and adding the user control to it, we can render the result as a string. To use the helper on a view, we can call it like this:

{% raw %}
```csharp
@(Html.RenderControl<MyCustomControl>("~/Test/MyCustomControl.ascx"))
```
{% endraw %}

The entire method call needs to be surrounded with parentheses, otherwise the angle brackets will confuse the Razor view engine. If the user control has properties that need to be set, we can call the method overload and pass it an expression:

{% raw %}
```csharp
@(Html.RenderControl<MyCustomControl>("~/Test/MyCustomControl.ascx", x => x.Name = "Malvin"))
 
@(Html.RenderControl<MyCustomControl>("~/Test/MyCustomControl.ascx", x =>
    {
        x.Name = "Malvin";
        x.Age = 25;
    }))
```
{% endraw %}
