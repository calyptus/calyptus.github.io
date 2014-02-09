---
layout: post
title: Why you shouldn't return false in MooTools event handlers
date: 2009-07-25 01:53:59.000000000 +02:00
categories:
  - seb
tags:
  - javascript
  - mootools
---
Let's say I have a link (anchor tag with href), and I wish to attach an event listener to it.
{% highlight xml %}
<ul>
  <li><a id="mylink" href="http://...">my link</a></li>
</ul>
{% endhighlight %}
{% highlight js %}
document.id('mylink').addEvent('click', function(){
  console.log('hello world');
});
{% endhighlight %}

Now, if I click the link it will log the message but the browser window will also visit the location of the link. There are a bunch of such default behaviors to pretty much every event in the DOM. If we're implementing custom behavior, we typically want to prevent this default behavior. A common practise is to have the method return false as such:

{% highlight js %}
document.id('mylink').addEvent('click', function(){
  console.log('hello world');
  return false;
});
{% endhighlight %}
**THIS IS BAD!** Don't. To understand the reason for this, you need to understand <a href="http://www.quirksmode.org/js/events_order.html">event bubbling</a> and the difference between <a href="http://mootools.net/docs/core/Native/Event#Event:preventDefault">preventDefault </a>and <a href="http://mootools.net/docs/core/Native/Event#Event:stopPropagation">stopPropagation</a>.

## Event bubbling and stopPropagation
When an event is dispatched, it first fires the listeners of the 'mylink' element (not quite true, but we don't use capture). But then it propagates (bubbles) up to the LI-element, UL-element, BODY-element etc.  So for every click on any element, the 'click' event is triggered on the BODY-element. After all of that, the default behavior of the browser is triggered.

*In most browsers bubbling continues to the document and window objects, but that's not always true for IE.*

This is a powerful model. It allows us to do things like <a href="http://www.clientcide.com/code-releases/event-delegation-for-mootools/">Event delegation</a>. You can place a listener on the UL-element to catch any events triggered on the LI-elements without adding listeners to all the existing or any new LI-elements.

Sometimes we don't want bubbling to occur. Let's say for example that I wanted to have a 'click' event handler on the UL-element that handles clicks on the UL area outside of any A-element. Then I could accept the <a href="http://mootools.net/docs/core/Native/Event">Event object</a> as the first parameter, use <a href="http://mootools.net/docs/core/Native/Event#Event:stopPropagation">stopPropagation</a> during the click event on the A-element to stop the event before it reaches the UL.

{% highlight js %}
document.getElements('ul').addEvent('click', function(){
  console.log('You clicked within the UL but outside of any link.');
});

document.getElements('a').addEvent('click', function(event){
  console.log('You clicked a link.');
  event.stopPropagation();
});
{% endhighlight %}

## preventDefault
In my example above the browser would still visit the href of the link. Stopping propagation (bubbling) doesn't actually prevent the default browser action. So we also need to call preventDefault during the click event to prevent the default operation of clicking a link.

{% highlight js %}
document.getElements('a').addEvent('click', function(event){
  console.log('You clicked a link.');
  event.stopPropagation();
  event.preventDefault();
});
{% endhighlight %}

Now since this is fairly common MooTools has a shortcut for doing both stopPropagation AND preventDefault. Namely the stop() method:

{% highlight js %}
document.getElements('a').addEvent('click', function(event){
  console.log('You clicked a link.');
  event.stop();
});
{% endhighlight %}

## So, why is return false bad?
In the standard browser DOM model it's **equivalent to calling `event.preventDefault();`** but **in MooTools it's equivalent to calling `event.stop();`** i.e. it also calls stopPropagation.

This is a problem. If you use this model routinely you may not notice that you actually prevent plugins attached to elements higher up in the bubbling chain.

Let's say I want to use the 'mouseleave' event to hide the UL-element when the mouse leaves. If I also return false on the 'mouseout' event on the A-element, I may not get the 'mouseleave' event because the A-element stops it. OR maybe I have a plugin higher up that requires that my events bubble. It'll be even more prevalent as more plugins makes use of <a href="http://www.clientcide.com/code-releases/event-delegation-for-mootools/">Event delegation</a>.

Therefore you need to be very explicit about when you stop propagation and not.

Second of all, the "return false" API doesn't make sense. The function isn't failing. It isn't canceled. In fact, it's canceling a DIFFERENT function.

Therefore you should ALWAYS be explicit by calling either event.preventDefault(), event.stopPropagation() or event.stop(); instead of relying on an implicit convention that differs between frameworks.

*Returning a false value is a relic from the old days when we only had a single listener per event.*

## Binding Parameters
Sometimes you need to bind parameters that you wish to pass to an event listener. A common practise is to use <a href="http://mootools.net/docs/core/Native/Function#Function:bind">bind</a>.

{% highlight js %}
document.getElements('a').addEvent('click', function(paramA, paramB){
  // do something with this, paramA and paramB
  return false;
}.bind(someObj, [objA, objB]));
{% endhighlight %}

In this case you can't accept an <a href="http://mootools.net/docs/core/Native/Event">Event object</a> since you've bound your parameters to other objects. In this case you can use <a href="http://mootools.net/docs/core/Native/Function#Function:bindWithEvent">bindWithEvent</a> to let the first parameter (the event object) get through, while binding the remaining parameters.

{% highlight js %}
document.getElements('a').addEvent('click', function(event, paramA, paramB){
  // do something with this, paramA and paramB
  event.stop();
}.bindWithEvent(someObj, [objA, objB]));
{% endhighlight %}

## $lambda(false)
*"But I don't want to type out all of that just to stop an event. I like $lambda(false) to easily block events."*

People sometimes use the $lambda method to create a function that returns false to easily stop an event without doing anything else: el.addEvent('click', $lambda(false));

So you need a method that does nothing other than accepts an Event object and calls preventDefault, stopPropagation or stop? Thanks to <a href="http://keetology.com/blog/2009/07/20/up-the-herd-ii-native-flora-and-fauna">MooTools generics</a> you can easily do that like this:
{% highlight js %}element.addEvent('click', Event.preventDefault); // OR...
element.addEvent('click', Event.stopPropagation); // OR...
element.addEvent('click', Event.stop);{% endhighlight %}

*For you that think "return false;" saves bandwidth... "e," and "e.stop();" is two bytes shorter.*

## Additional Event Listeners on the same Element
Neither preventDefault or stopPropagation or <a href="http://dean.edwards.name/weblog/2009/03/callbacks-vs-events/">even an error</a> prevents any additional handlers/listeners on the same element. So if you have two handlers listening to the same event, then both will be triggered regardless of the result of either function.

That should be true for all Events, even Class events. More on that in MooTools 2.0...
