---
layout: post
title: Convert a SVG Path to CSS Animation Keyframes
date: 2011-09-12 16:16:14.000000000 +02:00
categories:
  - seb
tags:
  - css
  - javascript
  - art
  - svg
---
**<a href="http://csspathanimation.calyptus.eu/">Click Here to Test The Tool</a>**

It's not easy to generate keyframes for <a href="http://www.w3.org/TR/css3-animations/">CSS3 Animations</a>. <a href="http://csspathanimation.calyptus.eu/">This new CSS Path Animation tool</a> can help you convert SVG paths into CSS keyframes.

## SVG
Using SVG you can use the <a href="http://www.w3.org/TR/SVG/animate.html#AnimateMotionElement">animateMotion</a> element to move and rotate content following along a complex curved path. However, some vendors have turned away from SVG for animation and prefer CSS as the way to do animations. For HTML content it tends to be very invasive. Using SVG with foreign objects tend to be poorly implemented as well.

## JavaScript
Most JavaScript frameworks for vector graphics (<a href="http://github.com/calyptus/art">including ART</a>) can do the same thing with JavaScript animations.

Some people think that CSS animations inherently provide better performance than JavaScript animations. This is not strictly true. It is the hardware compositing step that's the important part. You can use JavaScript animations together with requestAnimationFrame and 3D transforms to trigger hardware compositing. You get the same level of performance as with CSS.

However, this won't work in JavaScript-free environments, such as some banner ad sandboxes. CSS animations provide a nice separation of concerns and easy of use.

NOTE: If available, JavaScript is the preferred way to do this kind of animation efficiently. In a later post, I'll show you how to do this with <a href="http://github.com/calyptus/art">ART</a>.

Another aspect is that the JavaScript to run this animation can be very small.

## CSS
If you prefer <a href="http://www.w3.org/TR/css3-animations/">CSS Animations</a>, how do you create the keyframes to animate an object along a complex path?

If you only have straight lines, it is easy. Just apply a <a href="http://www.w3.org/TR/css3-2d-transforms/">CSS Transform</a> to the X and Y coordinates of each point along the path. The rotation can be adjusted based on the angle of the line if you need it.

## CSS Curve Animation
Things get much more complicated when you consider smooth curved paths. I.e. bezier curves or arcs.

The typical way to handle curves, is to subdivide them into very small straight line segments. However, if you generate this many CSS keyframes, the document would be huge. That's not an option. We need a better technique.

## The Arc Rotation Technique
For arcs, you can adjust the transform origin and rotate the element around the arc's center point. This is the optimal technique for simple arcs such as circles and ellipses. You can approximate bezier curves using several arc segments. This can generate very small files for simple curves, or very big files for complex curves. This means it's unreliable for an automated conversion tool.

If you want to avoid rotating the element and only move along a path, then you need an additional nested element to counter act the rotation.

## The Timing Function Technique
A future new best friend of mine pointed out to me that you can animate the X and Y axis using different easing functions. That way the timing will cause the animation to be a smooth curve. The CSS animation standard specifies a timing function based on a cubic bezier. This gives us a lot of flexibility to define an easing function.

The trick is in defining the timing function used. A simple implementation would specify bezier curve control points on the second and forth argument to the timing function, leaving the first and third argument at 0 and 1 respectively. This gives a perfect motion along the path. Unfortunately, it doesn't move at a constant speed.

To get the animation to move across the curve at a constant speed we have to use a little black magic math to approximate the motion at constant speed.

Not all curves can be animated using this technique due to limitations placed on the timing function. If a complex curve is found, we split it into two, until we have curves that can be animated this way. This provides us with a generic technique that can be used reasonably well for any path.

## Hardware Acceleration
The arc technique uses a single CSS transform and is therefore easily hardware accelerated.

The timing function technique, requires different easing functions for each axis. That means that each axis needs it's own animation.

Luckily we can apply multiple animations on a single element by animating the "left" and "top" properties independently. This is simple to use and non-invasive. However, this doesn't currently provide the hardware compositing with sub-pixel accuracy, like the "transform" property does.

Unfortunately, we can't animate each axis of a transform independently. The way to solve this is to use three nested elements and apply one transform for each element. This is a bit more invasive but provides the best visuals.

(UPDATE: The IE10 preview has a bug in nested transforms. At least in software mode. The previous method works well though and it's hardware accelerated in IE. If the bug remains, I'll update the converter to take this into account.)

## Conclusion
In <a href="http://csspathanimation.calyptus.eu/">the conversion tool</a> I use the timing function technique with three nested elements. This gives us a generic solution that is also hardware accelerated in current Webkit and Mozilla browsers.

The resulting CSS is very bloated. GZIP:ing does a lot to the size but it's still fairly big. There are a few action points browser vendors and W3C people could do to help:

1) Hardware accelerate left/top absolute positioning with sub-pixel precision. IE9 did this from the start. This is where the full(er) hardware pipeline of IE9 shines. This would allow us to use a single element for this hack.

2) Drop vendor prefixes on specs that are far along. CSS keyframes can be very bloated and they have to be multiplied in full for each vendor.

3) Standardize something like SVG's animateMotion, but for CSS. SVG isn't less verbose but there's a specialized element to handle this scenario. This same hack would be even more bloated in SVG.

A CSS rule could be used to name a SVG-style path. This name can then be referenced by the "animation-name" property to apply a transform to an element along a path. The current hack can be used to provide backward compatibility.

{% highlight css %}
#target {
  animation: myMotion 1s infinite;
}

@motion myMotion {
  path: 'm0,0 l100,100 ...';
}
{% endhighlight %}
I think I'll prototype something like this for LESS.js.

**UPDATE:**
I've started <a href="http://lists.w3.org/Archives/Public/www-style/2011Sep/0295.html">a discussion about standardizing this pattern on the www-style mailing list</a>.

The current idea is to add a "motion" property that's valid only within the context of keyframes.

{% highlight css %}
#target {
  animation: myMotion 1s infinite;
}

@keyframes myMotion {
  0% {
    motion: m0,0 l100,100 ...;
  }
  50% {
    motion: M100,100 L0,0 ...;
  }
}
{% endhighlight %}
