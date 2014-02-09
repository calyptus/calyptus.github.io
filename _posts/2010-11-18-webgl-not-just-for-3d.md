---
layout: post
title: WebGL - Not just for 3D
date: 2010-11-18 05:11:28.000000000 +01:00
categories:
  - seb
tags:
  - javascript
  - webgl
  - opengl
  - 3d
  - 2d
  - art
  - diffusion-curves
  - glsl
  - page-curl
---
**UPDATED: The demos are now updated to work around some bugs in the ANGLE implementation currently in Chrome 9 beta and Firefox 4 beta 8.**

Those of you who are <a href="http://twitter.com/sebmarkbage">following me on Twitter</a> know that I've been really getting into <a href="http://www.khronos.org/webgl/">WebGL</a> lately. Basically it's 1:1 JavaScript bindings to OpenGL ES 2.0 including GLSL (OpenGL Shading Language). This will allow you to access the power of GPU hardware directly in your website.

Safari, Chrome and Firefox all have WebGL enabled in their latest betas/nightly builds. Opera is on the working group as well. Additionally modern iOS and Android devices have OpenGL ES support so mobile support is probably not far away. Firefox already has WebGL support on Maemo.

## Microsoft is not yet on board
While Microsoft is constantly bragging about the fully hardware accelerated rendering in IE9 using DirectX. They're the only one of the major players not yet in the WebGL working group. WebGL is supposedly designed so that it can be implemented on top of DirectX. This allows for stronger implementations on Windows where OpenGL support is poor. Mozilla and Google are already <a href="http://news.softpedia.com/news/Google-s-ANGLE-Project-WebGL-Based-on-DirectX-137892.shtml">doing this</a> in their implementations.

Microsoft obviously have a lot invested in DirectX and may not want to support an API so close to the competing OpenGL. However, given their recent effort to support standards I'm hopeful they'll get on board for IE 10.

## Fallbacks
Given that no browser supports WebGL yet, other than in beta versions. We will obviously need some fallback.

<a href="http://mrdoob.com/">Mr. Doob</a> is doing some great progress on a 3D API with Canvas 2D and SVG fallback support - called <a href="https://github.com/mrdoob/three.js/">three.js</a>. However, I think this will be of limited use in most real-world scenarios.

For any proper 3D beyond gimmicks you'll want to use hardware acceleration. The exception being data visualization.

Flash and Silverlight already have pixel shader support. This doesn't help 3D much but they do help for 2D special effects etc. Although these are not hardware accelerated, they can run multi-core and they're faster than anything you can do with 2D canvas.

Additionally Adobe has announced hardware accelerated 3D support through the <a href="http://labs.adobe.com/technologies/flash/molehill/">Molehill APIs</a>. I'm sure Microsoft is not far behind with Silverlight. Once these get out to IE users, we'll have direct hardware capabilities on all the major platforms.

## Abstractions
WebGL is a horrible looking API. It's intentionally a direct translation of the C interface. The intension is that people will provide abstractions on top of it. E.g. a general game engine or a data visualization API built on top of it.

Some have tried to make thin wrappers around it. Like <a href="http://mootools.net">MooTools</a> or <a href="http://jquery.com">jQuery</a> wraps the DOM in a thin API abstraction. I think that is a mistake. You really need to optimize for performance and to do that you need a higher level abstraction that works directly to the WebGL API.

I have no intension of making a thin GL wrapper nor a single generic 3D API. Although...

## It's not just for 3D
Those of you <a href="http://github.com/calyptus">following me</a> on <a href="http://github.com/">GitHub</a> know that I've been doing a lot of work on the <a href="http://mootools.net/">MooTools</a> vector graphics library - <a href="http://github.com/calyptus/art">ART</a>. We already have great SVG and VML support. Next for me is to focus on high performance scenarios.

To start off, I've implemented two 2D use cases that are otherwise difficult to do properly using CPU only.

<a href="http://labs.calyptus.eu/diffusioncurves/">Diffusion Curves</a>

<a href="http://labs.calyptus.eu/diffusioncurves/"><img alt="" src="http://blog.calyptus.eu/wp-content/uploads/diffusion-curves.jpg" title="Diffusion Curves" class="alignnone" width="512" height="512" /></a>

A Diffusion Curve is a new vector primitive. To understand what they are and why they're incredibly cool, I recommend watching <a href="http://artis.imag.fr/Publications/2008/OBWBTS08/">the video from the authors of the original research paper</a>. Implementing a rasterizer of Diffusion Curves involves a diffusion and blur step that are both very computationally expensive. However, using a clever implementation you can take advantage of GPU hardware acceleration to get acceptable performance. There have been talks about introducing diffusion curves in SVG 2.0. But until that gets into browsers, we can already use it through the power of WebGL.

<a href="http://labs.calyptus.eu/pagecurl/">Page Curl Effect</a>

<a href="http://labs.calyptus.eu/pagecurl/"><img alt="" src="http://blog.calyptus.eu/wp-content/uploads/page-curl.jpg" title="Page Curl" class="alignnone" width="512" height="512" /></a>

A page curling effect can be implemented many different ways. However, to get a smooth realistic effect on dynamic content you really need some acceleration. This applies to many special effects that you can apply to 2D content.

Safari doesn't implement SVG filters yet. Such filters can be implemented in terms of WebGL and extended further to advanced blending effects and crazy transforms.

## Computation
WebGL is a great complement to JavaScript. JavaScript can be used to implement business logic and UI components while WebGL is used for computationally expensive tasks. Many of the use cases of NaCi (Native Client) goes out the window. You can even use WebGL to performance non-graphics tasks such as audio signal analysis/transformation.

Now all we need is a JavaScript based OpenCL like API for running GPU accelerated workers. There are already several projects doing <a href="http://mathema.tician.de/software/pyopencl">something similar</a> with Python.

## Learning more about WebGL
The best way to learn WebGL is to head over to <a href="http://learningwebgl.com/">Learning WebGL</a> and check out <a href="http://twitter.com/gpjt">Giles Thomas</a>' excellent introductory lessons.

## Coming Up
 * Lessons Learned - WebGL
 * Lessons Learned - Diffusion Curves
