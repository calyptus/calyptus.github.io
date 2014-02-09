---
layout: post
title: The Reactive Extensions for JavaScript - MooTools Integration
date: 2010-03-07 22:08:00.000000000 +01:00
categories:
  - seb
tags:
  - javascript
  - mootools
  - reactive-extensions
---
*This is a follow up to my earlier post about the <a href="http://blog.calyptus.eu/seb/2010/03/rx-for-javascript/">Reactive Extensions (Rx) for JavaScript</a> by Microsoft's DevLabs. This is also in response to Matthew Podwysocki's post on <a href="http://weblogs.asp.net/podwysocki/archive/2010/03/05/introduction-to-the-reactive-extensions-for-javascript-jquery-integration.aspx">jQuery integration</a> (which deserves some credit for putting it out there).*

*I will assume some familiarity with Rx.*

Just like any other DOM library, MooTools has a way of working with native and custom passive DOM events. We can easily give Element object and the Elements collection a method to provide these events as "Observables". In the jQuery example the method name "ToObservable" was added to the jQuery object, accepting an event type parameter, which was my initial reaction as well. But I'm going to call mine <strong>getEvent</strong> as in "<em>getting a stream of events given the event type</em>".

{% highlight js %}
var observableFromEvent = function(type){
  var self = this;
  return Rx.Observable.Create(function(observer){
	  var fn = function(event){
		  observer.OnNext(event);
	  };
	  self.addEvent(type, fn);
	  return function(){
		self.removeEvent(type, fn);
	  };
  });
};

Window.implement('getEvent', observableFromEvent);
Document.implement('getEvent', observableFromEvent);
Element.implement('getEvent', observableFromEvent);
Elements.implement('getEvent', observableFromEvent);
{% endhighlight %}

*These are infinite Observables but we could also make .destroy() trigger onComplete to make them finite as well.*

## Flickables Example
Instead of the canonical Drag and Drop example I thought I show a twist. Let's say we want to listen to a mouse flick. The mouse position have to move over 100px in 200ms. Then we want the angle of the flick.

{% highlight js %}
var angleFromPosition = function(position, center){
	var diffX = position.x - center.x, diffY = position.y - center.y;
	var distance = Math.sqrt(diffX * diffX + diffY * diffY);
	var angle = Math.atan2(diffY + distance, diffX) * 360 / Math.PI;
	return { distance: distance, angle: angle };
};

var distanceReached = function(angle){ return angle.distance > 100; };

var timeLimit = Rx.Observable.Timer(200);

var mousePositions = document.getEvent('mousemove')
					 .Select(function(event){ return event.page; });

var flicks = document.getElements('.flickable')
			 .getEvent('mousedown')
			 .SelectMany(function(event){
				 return mousePositions
					 .Select(angleFromPosition.bindWithEvent(null, event.page))
					 .TakeUntil(document.getEvent('mouseup'))
					 .TakeUntil(timeLimit)
					 .Where(distanceReached)
					 .Take(1);
			 });

// ...

flicks.Subscribe(function(current){
	console.log('Flicked in direction: ' + current.angle + '°');
});
{% endhighlight %}

## Events Mixin
MooTools has a very strong benefit compared to many other libraries. The publish/subscribe pattern is made explicit even for custom classes, using the Events mixin. By implement our "getEvent" method on this class we can use Rx on all custom MooTools classes that provide passive events.

{% highlight js %}
Events.implement('getEvent', observableFromEvent);
{% endhighlight %}

## Side-effects
Rx allows for the act of subscribing to an event to trigger an action/side-effect. Think of the Request object for example. You can use the act of subscribing to it, to issue a HTTP request. Then we can turn the subsequent events like success and failure into the Observable interface. This means that Request is a complete Observable in it self. This is what I was saving the conversion name <strong>toObservable</strong> for.

{% highlight js %}
Request.implement({

  toObservable: function(){
	var self = this;
	return Rx.Observable.create(function(observer){
	  
	  var listeners = {
	  
		success: function(result){
		  self.removeEvents(listeners);
		  observer.OnNext(result);
		  observer.OnCompleted();
		},
		
		cancel: function(){
		  self.removeEvents(listeners);
		  observer.OnCompleted();
		},
		
		failure: function(xhr){
		  self.removeEvents(listeners);
		  observer.OnError(xhr);
		}
	  
	  };

	  if (!self.running || self.options.link == 'cancel'){
		self.addEvents(listeners).send();
		return function(){
		  self.removeEvents(listeners).cancel();
		};
	  }
	  
	  if (self.options.link == 'chain'){
		var disposed, running;
		self.chain(function(){
		  running = true;
		  if (!disposed) self.addEvents(listeners).send();
		});
		return function(){
		  if (running) self.removeEvents(listeners).cancel();
		  disposed = true;
		};
	  }
	  
	  observer.OnComplete();
	  return function(){};
	  
	});
  }

});
{% endhighlight %}

This creates a finite stream of events - only one response to be exact. However, since the act of subscribing to it causes it to occur we can have it trigger repeatedly as part of a composite stream of events.

MooTools' Fx provides a similar concept but slightly different. Even though we don't get an event for each tick, we still get an asynchronous complete event. This means we can insert Fx as part of a composite stream of events.

Fx also requires from/to arguments to be passed at the start. So we add the option "defaultArgs" to allow us to pass those at initialization.

{% highlight js %}
Fx.implement({

  toObservable: function(){
	var self = this;
	return Rx.Observable.create(function(observer){
	  
	  var listeners = {
	  
		complete: function(){
		  self.removeEvents(listeners);
		  observer.OnCompleted();
		},
		
		cancel: function(){
		  self.removeEvents(listeners);
		  observer.OnCompleted();
		}
	  
	  };

	  if (!self.running || self.options.link == 'cancel'){
		self.addEvents(listeners).start.run(self.options.defaultArgs, self);
		return function(){
		  self.removeEvents(listeners).cancel();
		};
	  }
	  
	  if (self.options.link == 'chain'){
		var disposed, running;
		self.chain(function(){
		  running = true;
		  if (!disposed)
			self.addEvents(listeners).start.run(self.options.defaultArgs, self);
		});
		return function(){
		  if (running) self.removeEvents(listeners).cancel();
		  disposed = true;
		};
	  }
	  
	  observer.OnComplete();
	  return function(){};
	  
	});
  }

});
{% endhighlight %}

Of course since there are a lot of other classes extending the Request and Fx classes, you get the same benefits on them. This is one of the true benefits of MooTools' modular extensibility.

*That is one of the benefits of using the class(ical) pattern in JavaScript. More on that next time...*

## Side-effects Example
{% highlight js %}
var popup = document.id('popup');
var showPopup = new Fx.Morph(popup, { property: 'opacity', defaultArgs: 1 });
var feed = new Request.JSON({ url: 'mydata.json', method: 'get' });

var showFeed = feed.toObservable()
			   .Do(function(data){ popup.set('text', data); })
			   .Concat(showPopup.toObservable());

// ...

showFeed.Subscribe(); // loads mydata.json into #popup and displays it
{% endhighlight %}

## Using Arrays in Unit Tests
Since natives are allowed to be extended within the MooTools theorem, we can add a convenience method to turn an Array into an observable stream of content.

{% highlight js %}
Array.implement('toObservable', function(){return Rx.Observable.FromArray(this);});
{% endhighlight %}

We can use this to fake the "flicks" event stream in our earlier example. We avoid having to include complex asynchronous tests or user action tests.

{% highlight js %}
var flicks = [
		{ angle: 0, distance: 100 },
		{ angle: 45, distance: 100 },
		{ angle: 90, distance: 100 }
	].toObservable();

// Unit tests
// Synchronously testing code that's depending on a flick event stream
{% endhighlight %}

## Web Sockets and Web Workers
Now imagine this on a stream of events coming in from <a href="http://dev.w3.org/html5/websockets/">Web Sockets</a> or <a href="http://dev.w3.org/html5/workers/">Web Workers</a>.

You could set up a web socket to asynchronously feed you JSON objects, and easily hook that up to the rest of you UI just as easily as the Request example above.
