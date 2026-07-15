---
layout: post
title: "Loading animations using pure CSS"
date: 2013-12-26 14:55:53 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2013/12/26/loading-animations-using-pure-css/
---

With the advent of CSS animations, it's quite easy to create a loading animation using just CSS. Loading animations have traditionally been done using an animated gif. Using CSS animations only requires a single `div` element and a few lines of CSS:

{% raw %}
```css
#loading-image
{
	width: 25px;
	height: 25px;
	border-width: 8px;
	border-style: solid;
	border-color: #000;
	border-right-color: transparent;
	border-radius: 50%;
	animation-name: loading;
	animation-duration: 1s;
	animation-timing-function: linear;
	animation-iteration-count: infinite;
}

@keyframes loading
{
	0% { transform: rotate(0deg); }
	100%   { transform: rotate(360deg); }
}
```
{% endraw %}

Here is the result in [JSFiddle](http://jsfiddle.net/d3V8F/1/).

I recently participated in a code review for a website and instead of using animated images, a developer decided to use CSS animations. While this is neat, I believe it's a mistake to use this on a customer facing website. Perhaps my opinion will change in five years, but there are still too many people using older browsers that don't support CSS animations.

Using pure CSS does have some merit. For example, a page might load a *tiny* bit faster because there is one less image it has to download, which reduces the number of http requests and the size of the page. You can also use LESS to dynamically change the animation color to match customer defined themes, background colors, etc....

While there are some reasons to use CSS animations, there are more reasons not to. The most important reason against using CSS animations at this time is avoiding unnecessary complexity. If you decide to use CSS animations in customer facing websites, you'll still need to include a fallback method for browsers that don't support it. I don't see any reason to complicate things when animated gifs work perfectly fine.

Unless a website is highly dynamic with ever changing colors, I don't see a reason to use CSS animations for loading images. Again, my opinion might change in five years when more browsers support CSS animations.
