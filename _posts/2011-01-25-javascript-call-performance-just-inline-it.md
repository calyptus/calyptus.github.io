---
layout: post
title: JavaScript Call Performance - Just Inline It
date: 2011-01-25 00:46:49.000000000 +01:00
categories:
  - seb
tags:
  - javascript
  - performance
---
There are a couple of well-known micro optimization techniques when you need that extra speed within an expensive loop. It generally comes down to eliminating function calls and flattening the closure scope.

Intuitively you don't want to bloat your download size with repeated code. You also want a clear separation of concerns to make your code understandable and managable. You want to separate common code into sub-routines (functions).

## Intra-object method calls
I've done <a href="http://jsperf.com/classes-with-inner-function-calls-instantiation">a number</a> <a href="http://jsperf.com/classes-with-inner-function-calls">of tests</a> <a href="http://jsperf.com/classes-with-many-inner-function-calls">on repeated calls</a> within a single object. This is typically where an expensive loop would occur.

Now you might think that property look-ups are expensive and should be avoided. Since in a naive implementation they would be continued hash-table look-ups. However, traversing closure scopes and .call(bind) invocations tend to be more expensive in real world engines. Generally, you are better off invoking the function, by property name, on the same prototype - continuously. Engines optimize for this pattern. In the tests you can see that this is even more prevalent in recent optimizations.

## Super/base method calls
In Class-oriented/OOP patterns, you often see the need to override an inherited method. However, while doing so you wish to call the overridden (super/base/parent) function from within the new one.

The most common way of doing this is by storing a reference to the prototype object, then invoking the function using .call() or .apply() to set the "this" context to the right value. However, as I've shown, navigating a closure and invoking the function using .call() is more expensive than invoking prototype methods.

The benefit of storing a reference to the prototype is that you get a live property that can be monkey patched with a new super function on the fly. However, relying on this pattern can be tricky.

You're better off having a module system that supports proper file ordering of monkey patches. Where monkey patches are applied before any child modules inherit from their parent. This will enable better optimization on engines that depend on declaration order to optimize their hidden classes (like V8). It also enables monkey patching of mixins that are copied. I.e. has no live inheritance (since JS doesn't support multiple inheritance).

If we can assume that we don't need on-the-fly updates of super functions, then we can optimize this further. Again, <a href="http://jsperf.com/super-call">my tests</a> show, that it's faster to invoke a method on the same prototype than finding a function reference and invoking .call().

I've toyed with some <a href="https://gist.github.com/790455">syntax experiments</a> to make this prettier.

## Just Inline It
<a href="http://twitter.com/steida">Daniel Steigerwald</a> pointed out that once you make the assumption that we don't need on-the-fly monkey patches, you should just inline the code. This is always faster.

Let's take a look at some nonsensical code. The function bar calls the function foo four times.

{% highlight js %}
function foo(state){
	var my = 'code';
	for (var i = 0, l = my.length; i < l; i++)
		if (i % 10 == 2)
			state.push(my[i]);
		else if (i == 1)
			state.push(my[0]);
		else if (l == 2 && i == 3)
			state.push('foo');
		else
			state.push('else');
	return my;
}

function bar(condition, state){

	var complexCondition = !condition && state.length == 3;
	
	var value = state[0];

	while (condition){
		value = foo(state);
		if (complexCondition){
			state.push('someData');
			value = foo(state);
		} else if (value.length == 5){
			value = foo([]);
		} else if (value == 'else'){
			value = foo(state)
			state.push('test');
		}
		condition = (value == 'foo');
	}

}
{% endhighlight %}

This file weighs in at about 300 bytes gzipped.

If we instead inline (copy) the foo code into all four places in the bar function. We get something like this.

{% highlight js %}
function bar(condition, state){

	var complexCondition = !condition && state.length == 3;
	
	var value = state[0];

	while (condition){
			var stack = state;
			var my = 'code';
			for (var i = 0, l = my.length; i < l; i++)
				if (i % 10 == 2)
					stack.push(my[i]);
				else if (i == 1)
					stack.push(my[0]);
				else if (l == 2 && i == 3)
					stack.push('foo');
				else
					stack.push('else');
			value = my;
		if (complexCondition){
			state.push('someData');
			var stack = state;
			var my = 'code';
			for (var i = 0, l = my.length; i < l; i++)
				if (i % 10 == 2)
					stack.push(my[i]);
				else if (i == 1)
					stack.push(my[0]);
				else if (l == 2 && i == 3)
					stack.push('foo');
				else
					stack.push('else');
			value = my;
		} else if (value.length == 5){
			var stack = [];
			var my = 'code';
			for (var i = 0, l = my.length; i < l; i++)
				if (i % 10 == 2)
					stack.push(my[i]);
				else if (i == 1)
					stack.push(my[0]);
				else if (l == 2 && i == 3)
					stack.push('foo');
				else
					stack.push('else');
			value = my;
		} else if (value == 'else'){
			var stack = state;
			var my = 'code';
			for (var i = 0, l = my.length; i < l; i++)
				if (i % 10 == 2)
					stack.push(my[i]);
				else if (i == 1)
					stack.push(my[0]);
				else if (l == 2 && i == 3)
					stack.push('foo');
				else
					stack.push('else');
			value = my;
			state.push('test');
		}
		condition = (value == 'foo');
	}

}
{% endhighlight %}

This file weighs about 290 bytes gzipped. Wait, what? We more than doubled the original file size but the result is smaller?

Yes, the GZIP compression uses the <a href="http://en.wikipedia.org/wiki/LZ77">LZ77</a> algorithm to find repeated byte sequences. This means that inlining your code may actually result in smaller download sizes since some plumbing code is removed.

Is it cheating to compare gzipped file size instead of originals? No. You should always send your static uncompressed content using the GZIP compression. It's cheap. This is the real world baseline. You should optimize for this.

There are some quirks to look out for. Some minifiers rename variables inconsistently. This may result in inconsistent sequences. You should look out for this, since that would cause a larger file size.

The larger code base may cause a slightly slower start up, since the JS engine needs to parse and compile more code. This should be very very minimal though. Remember, we're optimizing for tight loops here.

## JS-to-JS Compilers
Our inlined code is ugly and unmaintainable. Luckily there are tools that can inline pretty source code for us. E.g. the <a href="http://closure-compiler.appspot.com">Google Closure Compiler</a>. Unfortunately the Closure Compiler doesn't seem to inline functions that are called more than once. Neither does it use these optimization techniques for super calls. Perhaps there are settings that I'm not aware of.

Recursive functions may be difficult to inline but there are tools that can rewrite these and do tail call optimizations as well.

However, this is just a hint of the awesomeness that could be achieved by using a custom JS-to-JS compiler. More on that later.
