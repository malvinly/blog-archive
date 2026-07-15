---
layout: post
title: "Understanding multiple anti-forgery tokens in ASP.NET MVC"
date: 2017-06-10 05:30:33 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2017/06/10/understanding-multiple-anti-forgery-tokens-in-asp-net-mvc/
---

The MVC helper "Html.AntiForgeryToken()" can be used to protect your application against [cross-site request forgery (CSRF)](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)). This will generate both a hidden field and a cookie that contains matching values that are validated on the server.

Our website utilizes multiple forms on the same view and each form contains an anti-forgery token. However, each call to "Html.AntiForgeryToken()" generates a different value. For example, multiple calls such as:

{% raw %}
```html
<html>
    <head></head>
    <body>
        @Html.AntiForgeryToken()
        @Html.AntiForgeryToken()
        @Html.AntiForgeryToken()
    </body>
</html>
```
{% endraw %}

... will generate multiple hidden fields that look like this:

{% raw %}
```html
<input name="__RequestVerificationToken" type="hidden" value="iAdQj5D0qrMuTggD8WpnOZPlVOfHg_qmPIEjnULAYd1h56cV2cL51rcaY8_UgxQbav5_6KTAtyE52ir1X6GmaS9ZPgw1" />
<input name="__RequestVerificationToken" type="hidden" value="Shvi8Bxe6-a8zfCfDGnxkaC-IETsbjkR9iIylwn-2VRWQ-GtQkdowdFw1biU7dN3j-xPJZHYQPe-hNfWspYjy_ZcCCY1" />
<input name="__RequestVerificationToken" type="hidden" value="ZhaVFngUMLo88jmTIx___BTWlYFyKh1GalwEeffRl0-o3Gu7_m98k6aQjO7IysZIdXxVx6TqL6QIfX19Uwq3Ia6dghA1" />
```
{% endraw %}

I wanted to learn how it works under the cover, so I used ReSharper and dotPeek to decompile the code to understand it better.

At the heart of the helper "Html.AntiForgeryToken()" is the method "GetFormInputElement" used to generate the hidden field.

{% raw %}
```csharp
public TagBuilder GetFormInputElement(HttpContextBase httpContext)
{
	this.CheckSSLConfig(httpContext);

	AntiForgeryToken cookieTokenNoThrow = this.GetCookieTokenNoThrow(httpContext);
	AntiForgeryToken newCookieToken;
	AntiForgeryToken formToken;
	this.GetTokens(httpContext, cookieTokenNoThrow, out newCookieToken, out formToken);

	if (newCookieToken != null)
		this._tokenStore.SaveCookieToken(httpContext, newCookieToken);

	if (!this._config.SuppressXFrameOptionsHeader)
		httpContext.Response.AddHeader("X-Frame-Options", "SAMEORIGIN");

	TagBuilder tagBuilder = new TagBuilder("input");
	tagBuilder.Attributes["type"] = "hidden";
	tagBuilder.Attributes["name"] = this._config.FormFieldName;
	tagBuilder.Attributes["value"] = this._serializer.Serialize(formToken);
	return tagBuilder;
}
```
{% endraw %}

To understand why different values are generated in the hidden fields, we need to first take a look at the internal class "AntiForgeryToken" that represents the verification token.

{% raw %}
```csharp
internal sealed class AntiForgeryToken
{
	private BinaryBlob _securityToken;

	public BinaryBlob SecurityToken
	{
		get
		{
			if (this._securityToken == null)
				this._securityToken = new BinaryBlob(128);

			return this._securityToken;
		}
		set
		{
			this._securityToken = value;
		}
	}

	// Removed other properties, fields, and methods.
	// ...
	// ...
}
```
{% endraw %}

The token is represented by a "BinaryBlob". Digging down into this class shows it uses "RNGCryptoServiceProvider" to randomly generate a 16 byte array.

{% raw %}
```csharp
private static byte[] GenerateNewToken(int bitLength)
{
	byte[] data = new byte[bitLength / 8];
	BinaryBlob._prng.GetBytes(data);
	return data;
}
```
{% endraw %}

Looking back at the "GetFormInputElement" method, we can see the code checks for the existence of a token from the cookies collection using the method "GetCookieTokenNoThrow". Drilling into the code shows it's not complicated. If it exists, it deserializes into an "AntiForgeryToken", otherwise it returns null.

{% raw %}
```csharp
public AntiForgeryToken GetCookieToken(HttpContextBase httpContext)
{
	HttpCookie cookie = httpContext.Request.Cookies[this._config.CookieName];

	if (cookie == null || string.IsNullOrEmpty(cookie.Value))
		return (AntiForgeryToken) null;

	return this._serializer.Deserialize(cookie.Value);
}
```
{% endraw %}

So what does it do with this deserialized token? We can see it's passed to the method "GetTokens". Drilling into this method eventually leads to the method "GenerateFormToken".

{% raw %}
```csharp
private void GetTokens(HttpContextBase httpContext, AntiForgeryToken oldCookieToken, out AntiForgeryToken newCookieToken, out AntiForgeryToken formToken)
{
	newCookieToken = (AntiForgeryToken) null;

	if (!this._validator.IsCookieTokenValid(oldCookieToken))
		oldCookieToken = newCookieToken = this._validator.GenerateCookieToken();

	formToken = this._validator.GenerateFormToken(httpContext, AntiForgeryWorker.ExtractIdentity(httpContext), oldCookieToken);
}

// From the validator's class
public AntiForgeryToken GenerateFormToken(HttpContextBase httpContext, IIdentity identity, AntiForgeryToken cookieToken)
{
	AntiForgeryToken antiForgeryToken = new AntiForgeryToken()
	{
		SecurityToken = cookieToken.SecurityToken,
		IsSessionToken = false
	};

	// Removed some code related to identities and additional data.
	// ...
	// ...

	return antiForgeryToken;
}
```
{% endraw %}

So from what we can see, if a token can be deserialized from the request's cookie collection, it'll reuse that token instead of generating a new one. If a token doesn't exist in the cookie collection, it'll instantiate a new instance of "AntiForgeryToken" and randomly generate a new 16 byte array to represent the token.

Going back to the method "GetFormInputElement", we can see it calls the method "SaveCookieToken" after generating or reusing the existing token.

{% raw %}
```csharp
public void SaveCookieToken(HttpContextBase httpContext, AntiForgeryToken token)
{
	HttpCookie cookie = new HttpCookie(this._config.CookieName, this._serializer.Serialize(token))
	{
		HttpOnly = true
	};

	if (this._config.RequireSSL)
		cookie.Secure = true;

	httpContext.Response.Cookies.Set(cookie);
}
```
{% endraw %}

After generating the first token and saving it to the cookie collection, all subsequent calls to the helper method "Html.AntiForgeryToken()" will follow the same steps and reuse the existing token from the cookie collection instead of generating a new value. Since it is a session cookie, this means the anti-forgery token's value is generated only once during a browser session and is reused for all subsequent calls.

So why are the hidden field values different from one another if they are reusing the same token? To answer that, we have to look at the token serializer.

{% raw %}
```csharp
public string Serialize(AntiForgeryToken token)
{
	using (MemoryStream memoryStream = new MemoryStream())
	{
		using (BinaryWriter binaryWriter = new BinaryWriter((Stream) memoryStream))
		{
			binaryWriter.Write((byte) 1);
			binaryWriter.Write(token.SecurityToken.GetData());
			binaryWriter.Write(token.IsSessionToken);

			// Removed some code related to identities and additional data.
			// ...
			// ...

			binaryWriter.Flush();
			return this._cryptoSystem.Protect(memoryStream.ToArray());
		}
	}
}
```
{% endraw %}

The "Protect" method uses the internal class "AspNetCryptoServiceProvider" to encrypt the token using our specified machineKey in the config. So while the encrypted values may look different, the decrypted values are the same. To test this, we can use the decompiled code from dotPeek to "Unprotect" the encrypted values.

{% raw %}
```csharp
byte[] one = MachineKey45CryptoSystem.Instance.Unprotect("iAdQj5D0qrMuTggD8WpnOZPlVOfHg_qmPIEjnULAYd1h56cV2cL51rcaY8_UgxQbav5_6KTAtyE52ir1X6GmaS9ZPgw1");
byte[] two  = MachineKey45CryptoSystem.Instance.Unprotect("Shvi8Bxe6-a8zfCfDGnxkaC-IETsbjkR9iIylwn-2VRWQ-GtQkdowdFw1biU7dN3j-xPJZHYQPe-hNfWspYjy_ZcCCY1");
byte[] three = MachineKey45CryptoSystem.Instance.Unprotect("ZhaVFngUMLo88jmTIx___BTWlYFyKh1GalwEeffRl0-o3Gu7_m98k6aQjO7IysZIdXxVx6TqL6QIfX19Uwq3Ia6dghA1");
```
{% endraw %}

Comparing all three byte arrays reveals they are identical.

In summary, the verification tokens generated from "Html.AntiForgeryToken()" are all identical within a browser session, regardless how many times we call it. The values appear different because they're encrypted using our machineKey.
