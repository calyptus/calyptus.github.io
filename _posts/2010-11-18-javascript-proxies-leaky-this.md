---
layout: post
title: JavaScript Proxies - "Leaky this"
date: 2010-11-18 00:31:17.000000000 +01:00
categories:
  - seb
tags:
  - javascript
---
<em>You can introduce AOP interception patterns by either changing the original instance or by using additional proxy objects. The later can introduce confusion among instances if not treated correctly. Skip to "<a href="#leaky-this">Leaky this</a>" if you're already familiar with proxies in ECMAScript Harmony.</em>

## Rewriting vs. Wrapping
Intercepting normal program flows using aspect oriented and other meta-programming patterns can be used for a variety of advanced tricks. Object-relational mapping, logging, security, customized APIs etc.

You can do this by rewriting the methods of the original class/prototype/instance, effectively providing a single instance. You can also do this by providing wrapper objects. That way you have two or more instances.

Rewriting instances or prototypes is easy to do in JS by overriding the functions on existing objects with your own wrapper functions. Although it's not always possible on the magical host objects. It can also be a problem when these objects have different requirements in various contexts. Such as in a secured sandbox vs. host environment.

Wrappers allow you to leave the original object intact while providing the wrapper to a specific context as if it's the real object.

## Wrappers in ECMAScript 5
In <a href="http://www.ecma-international.org/publications/files/ECMA-ST/ECMA-262.pdf">ECMAScript 5</a> you can already do seemingly perfect wrappers by getting all the properties of an object. Then you define those properties on the wrapper - called a proxy. These are custom getters and setters that delegate to the original object - called the subject.

{% highlight js %}
var subject = new Node();

var proxy = Object.create();

Object.getOwnPropertyNames(subject).forEach(function(name){
	Object.defineProperty(proxy, name, {
		get: function(name){
			// interception code
			return subject[name];
		}
	}
});
{% endhighlight %}

(For simplicity I've left out the prototype, setters, enumerations rules etc. Getters are enough to illustrate my point.)

These are fine for frozen objects. The problem is that they don't represent later changes to the subject. E.g. adding or deleting properties. Additionally, property changes applied to the proxy aren't propagated back into the subject.

## Proxies in ECMAScript Harmony
It looks like the next version of ECMAScript will get support for <a href="http://wiki.ecmascript.org/doku.php?id=harmony:proxies">proxy object</a> that can delegate all operations dynamically - using so called "catch-all" functions.

{% highlight js %}
var subject = new Node();

var proxy = Proxy.create({
	get: function(receiver, name){
		// interception code
		return subject[name];
	}
});
{% endhighlight %}

When we're reading a property from the proxy instance, our get-function gets called with the receiver (the proxy instance itself) and the name of the property that should be loaded. We delegate this request to the subject by reading the same property. We can insert any intercepting code as we choose.

This is more efficient and allows for dynamic evaluation of intercepting code. Many ORM patterns - such as <a href="http://en.wikipedia.org/wiki/Active_record">Active Record</a> - often use this technique in dynamic environments.

<a name="leaky-this"></a>**"Leaky this"**

However, a problem occurs when these two instances get confused. Imagine this scenario:

{% highlight js %}
var Node = function(){
  var self = this, children = [];
  self.addChild = function(child){
	children.push(child);
	child.parent = self;
	return self;
  };
};
{% endhighlight %}

If we wrap an instance of Node in a proxy. Then call:

{% highlight js %}
proxy.addChild(child).addChild(childOther);
{% endhighlight %}

We first call addChild through the proxy instance. Our child.parent property is set to the subject instance. The subject instance is returned and we next call addChild directly on the subject instance. Bypassing the proxy.

This is a well-known problem in languages that strongly binds the "this" keyword to a specific instance. <a href="http://rogeralsing.com/">Roger Alsing</a> calls this "<a href="http://www.puzzleframework.com/forum/thread.aspx?Thread=828">leaky this</a>" which I find to be an appropriate term since the abstraction is leaking.

However, imagine this JavaScript code:

{% highlight js %}
var Node = function(){
  this.children = [];
};

Node.prototype.addChild = function(child){
  this.children.push(child);
  child.parent = this;
  return this;
};
{% endhighlight %}

The addChild function now references the "this" keyword. If we repeat the same call in this scenario we get a different result. We call addChild while passing the proxy as the "this" reference. We set the parent property of child to the proxy instance. The proxy instance is returned, and the process is repeated on the second call. This shows the beauty of not having the "this" keyword bound to specific instances.

Although, sometimes you don't want to use the proxy instance in internal functions. You could use this hybrid to use the subject for whatever use internally, while exposing any proxies externally:

{% highlight js %}
var Node = function(){
  var self = this, children = [];
  self.addChild = function(child){
	children.push(child);
	child.parent = self;
	return this;
  };
};
{% endhighlight %}

## Conclusion
<a href="http://brendaneich.com/">Brendan Eich</a> <a href="http://brendaneich.com/2010/11/proxy-inception/">points out</a> that unaware consumers of a proxy should be able to treat it just as any other object. This is only true if the provider of both the proxy and the subject have carefully considered the implications of two instances and made sure that no leaking occurs. Code that can be wrapped needs to carefully consider what instance it is currently working with.

Not all code can be wrapped using a proxy indiscriminately.
