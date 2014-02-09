---
layout: post
title: The Reactive Extensions for JavaScript - Event Composition
date: 2010-03-06 15:37:22.000000000 +01:00
categories:
  - seb
tags:
  - javascript
  - reactive-extensions
---
I've been following the work on the <a href="http://msdn.microsoft.com/en-us/devlabs/ee794896.aspx">Reactive Extensions for .NET</a> (Rx) by <a href="http://research.microsoft.com/en-us/um/people/emeijer/ErikMeijer.html">Eric Meijer</a> and others over at Microsoft. At first look I was intrigued but didn't really understand the purpose of it. However, at a second look, I realized that it had the potential to solve every major problem I've had with advanced UI development in JavaScript.

## Asynchronous Programming - Composable Events
Modern UI development forces us to use asynchronous patterns for user actions, animations and data load. But to make UI development easy you can think of each operation as sequential. You can even artificially lock down the user interface - by disabling or hiding UI elements - while an operation is occurring.

If you want to optimize your user experience, you will need to start dabble in the complicated art of event composition. The problem occurs when you have complex interactions that depend upon other interaction or state.

You can solve this using various state machine patterns. However, I think you will find it quite difficult at times. Even if you do solve it, it's probably going to be for a specific purpose which is not easily generalizable nor extensible.

Various tools have tried using patterns like Futures and Promises. I think those patterns need to be applied at the language level to be really useful though.

## Reactive Programming in an Object Oriented World
JavaScript has introduced the map/filter/reduce methods on arrays to allow collection operations using a sequenced composition of functions.

There's one minor thing that JavaScript developers should note. In <a href="http://linqjs.codeplex.com/">LINQ</a> these operations are lazy iterables. The map/filter operations aren't actually executed until an .each() starts iterating over them. This avoids having to create duplicates of the result in memory. It also means that the underlying array can change after we call filter and map. This is similar to "live" collections in the DOM. But they can also be infinite in length just like <a href="https://developer.mozilla.org/en/Core_JavaScript_1.5_Guide/Iterators_and_Generators">Mozilla's Iterators</a>. The .each() call is still essentially a synchronous operation though.

<a href="http://channel9.msdn.com/shows/Going+Deep/Expert-to-Expert-Brian-Beckman-and-Erik-Meijer-Inside-the-NET-Reactive-Framework-Rx/">Erik Meijer and team</a> simply decided to make that iteration execution asynchronous.

This means that the source data can be asynchronous. So instead of thinking of Events as independent, think of them as a stream of data with an unknown length... (or an asynchronous list/array).

This means that you can now apply the same type of function composition to streams of events. Enter the <a href="http://msdn.microsoft.com/en-us/devlabs/ee794896.aspx">Reactive Extensions for .NET</a>.

Supposedly this could solve the problem of Event composition in the UI space.

## The Reactive Extensions for JavaScript
To my surprise, <a href="http://weblogs.asp.net/podwysocki/">Matthew Podwysocki</a> recently started a <a href="http://weblogs.asp.net/podwysocki/archive/2010/02/16/introduction-to-the-reactive-extensions-to-javascript.aspx">blog series</a> about the Reactive Extensions for JavaScript (also Microsoft to be clear). Apparently the benefits of this tool in the JS world has not gone unnoticed.

There are no bits officially released yet. However, considering <a href="http://weblogs.asp.net/podwysocki/">Matthew Podwysocki's</a> recent posts and <a href="http://live.visitmix.com/MIX10/Sessions/FTL01">Eric Meijer's upcoming talk</a> at the Mix conference... I wouldn't be surprised if something was released at Mix on March 17th.

## Learning More
I have since been interested in learning more about various alternative models. There's a research project called <a href="http://www.cs.umd.edu/~jfoster/arrows.pdf">Arrows</a> which provides a different model that's more purely functional. There's also a framework called <a href="http://www.flapjax-lang.org/">Flapjax</a> which is more of a DOM library aiming to provide reactive concepts to JavaScript.

To learn more about the Reactive Extensions, take a look at the <a href="http://channel9.msdn.com/tags/Rx/">videos about the .NET version posted by the team over at Channel 9</a>.

## Concerns
I'm not sure the first implementation of the Reactive Extensions is going to be the one to solve all these problems.

I think that many developers will have a difficult time thinking about these concepts in the terms of event streams. That could make it difficult to use the current method naming. In this sense, I think <a href="http://www.cs.umd.edu/~jfoster/arrows.pdf">Arrows</a> might be easier to get started with. It will allow you to think about events as sequential operations. However, I also see benefit in the model employed by the Reactive Extensions, IF we can all wrap our heads around it.

Another issue is the "Let" method. This may be difficult to know when to use for many developers. That's true even for LINQ. However, in the Reactive Extensions I have a feeling those issues will become even more prevalent. Hopefully there will be better syntactical sugar to solve this issue.

Rx has yet to prove itself in real world complex applications that goes well beyond single subscription examples. I may try to extend the canonical drag and drop examples to my own <a href="http://github.com/calyptus/mootools-draggable">HTML5 based Drag and Drop</a> model and plugins to stress test it.

## Naming Conventions
There's also the issue of upper camel case in method names. <a href="http://linqjs.codeplex.com/">LINQ for JavaScript</a> is also using this convention. I'm guessing they're trying to be compatible. However, the JavaScript convention is to use lower camel case method names, which also the <a href="http://ajax.codeplex.com/">new ASP.NET AJAX</a> library is doing. So I don't understand why.

In dynamic languages with limited auto-completion (IntelliSense) support, naming conventions are very important to follow.

Although, I do like the names Select/Where/OrderBy better than map/filter/sort since given the arguments, that tends to read better as a grammatical sentence.

## UPDATE: Event DSLs
I should mention that the MooTools 2.0 team has been working on a DSL based on the CSS selector syntax. This is an extension of the <a href="http://mootools.net/docs/more/Element/Element.Delegation">Element.Delegation</a> plugin.

The idea is to use event names and pseudos in combination to create custom composite event listeners. This example would listen to the first click event:

{% highlight js %}
element.addEvent('click:flash', firstClick);
{% endhighlight %}

This could enable a lot more powerful combinations of custom events. However, it doesn't enable passing of parameters and the composability of Rx and Arrows.
