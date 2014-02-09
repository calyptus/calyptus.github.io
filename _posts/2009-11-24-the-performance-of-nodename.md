---
layout: post
title: The Performance of .nodeName
date: 2009-11-24 06:43:01.000000000 +01:00
categories:
  - seb
tags:
  - javascript
  - performance
  - dom
---
I was researching various options of traversing nodes for <a href="http://github.com/subtleGradient/slick">Slick</a> and the <a href="http://github.com/calyptus/mootools-more/tree/range/Source/Native/">DOM Range for MooTools</a>. I realized that the nodeName property is incredibly slow to access in WebKit browsers. This is because it is working with qualified names (with namespaces and stuff) internally.

{% highlight js %}
if (node.nodeName == 'A') // do something with anchor tag
{% endhighlight %}

If you add case insensitive matching to that it will be even slower.

Instead I decided to try to check the constructor of the node to determine what type it is. For example for the anchor (A) tag, modern browsers will use the prototype of HTMLAnchorElement. This can potentially speed up these checks if you're looking for a known node type.

{% highlight js %}
if (node.constructor === HTMLAnchorElement) // do something
// OR...
if (node instanceof HTMLAnchorElement) // do something
{% endhighlight %}

I ran <a href="http://labs.calyptus.eu/NodeName/performance-test.html">this performance test</a> in various browsers. It traverses all nodes in a large HTML documents and checks which ones are anchor nodes. It first does a blank run to eliminate any initialization quirks. Then it does a control run without the anchor check. Then it tests each of the above models.

**IE6 and IE7** will obviously fail since they don't support the HTMLAnchorElement constructor/prototype. For that case you would have to fall back to the nodeName property.

**IE8** will be slightly slower with the constructor check than the nodeName check. But the difference is marginal in the overall scope of IE's slowness.

**WebKit** will gain significant performance using the constructor check. The difference is relatively small to the overhead of manually walking the tree. However, if you take the control value from the blank run into account, the difference of just the node type checks will be significant (several times faster). The slow part is the WebKit DOM API, so you will see this with both JavaScriptCore and V8 (Safari and Chrome respectively).

**Firefox** will be slower on the first run for some weird reason. But in subsequent runs the constructor check will be faster than the nodeName check.

*As a side note, node.tagName is no different. That is just an alias for node.nodeName.*

*In <a href="http://ejohn.org/blog/nodename-case-sensitivity/">John Resig's case sensitivity</a> he discusses the case inconsistencies of the nodeName property in various contexts and the impact on performance. For example, in IE, the value of nodeName of unknown elements (like the new HTML5 elements) keeps it original case as in the markup.*

*This means that any proper CSS selector search for such elements would have to run a case-insensitive match against the nodeName property. Unfortunately the little trick I've shown above doesn't remedy this problem because unknown elements will be lacking a known constructor. However, known Elements can still utilize this trick as a slight performance boost, while letting unknown element fallback to a case insensitive match.*

*UPDATE: I added a case-insensitive match to the <a href="http://labs.calyptus.eu/NodeName/performance-test.html">performance tests</a> using regular expressions - showing the added overhead compared to constructor checking.*
