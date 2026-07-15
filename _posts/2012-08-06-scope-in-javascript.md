---
layout: post
title: "Scope in JavaScript"
date: 2012-08-06 07:27:40 +0000
categories: ["General"]
original_url: https://nivlam.wordpress.com/2012/08/06/scope-in-javascript/
---

One of the things I commonly see confuse C# developers when they start working with JavaScript is how different scoping is compared C#. The code below will alert "2" twice. The equivalent C# code would cause a compile time error because "num" already exists in the current scope.

{% raw %}
```javascript
var num = 1;

if (num == 1) {
	var num = 2;
	alert(num);
}
	
alert(num);
```
{% endraw %}
Because JavaScript has function-level scope, blocks do not create a new scope. When we do require block-level scope, we may use an anonymous function to create a new scope.

{% raw %}
```javascript
var num = 1;

if (true) {
	(function() {            
		var num = 2;
		alert(num);
	})();
}

alert(num);
```
{% endraw %}

The code below will alert "2" once. The reason is because the variable declaration for "num" is actually moved to the top of the function scope.

{% raw %}
```javascript
var num = 1;

(function() { 
	if (num) { 
		alert(num);
	}

	var num = 2;
	alert(num);
})();
```
{% endraw %}

This code is functionally equivalent to:

{% raw %}
```javascript
var num = 1;

(function() { 
	var num;

	// "num" is undefined at this point.
	if (num) { 
		alert(num);
	}

	num = 2;
	alert(num);
})();
```
{% endraw %}

Variable declarations are moved to the top, even if they are unreachable or dead code. This code will alert "2" then "1":

{% raw %}
```javascript
var num = 1;

(function() {
	if (!num) {
		num = 2;
	}
	
	alert(num);
	
	return;
	var num = 99;
})();

alert(num);
```
{% endraw %}

If we removed the variable declaration in the inner scope, this code will alert "1" twice:

{% raw %}
```javascript
var num = 1;

(function() {
	if (!num) {
		num = 2;
	}
	
	alert(num);
	
	return;
	//var num = 99;
})();

alert(num);
```
{% endraw %}
