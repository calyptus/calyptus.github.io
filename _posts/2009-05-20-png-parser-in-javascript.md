---
layout: post
title: Parsing Base64 Encoded Binary PNG Images in JavaScript
date: 2009-05-20 23:39:53.000000000 +02:00
categories:
  - seb
tags:
  - javascript
  - base64
  - png
  - proof-of-concept
---
<em>The other day <a href="http://www.davidwalsh.name/">David Walsh</a> was experimenting with rendering images in the browser using regular tags as pixels. <a href="http://mad4milk.net/">Valerio</a> picked up the idea and made some enhancements. A server-side script transformed PNG files into a JSON image format for easy parsing on the client. That raised the question... How difficult would it be to do that parsing on the client instead?</em>

Why PNG? Well, other than becoming the new defacto standard for graphics it's a very simple format. It's also free of patents and uses only simple well known techniques. It makes it very easy to work with. This post is about parsing raw PNG image data in pure JavaScript. It has nothing to do with built in browser support for the format.

<strong>Base64 Encoding</strong>

JavaScript doesn't allow us to work with binary data directly. Even with XHR we can't work with the raw binary data because JavaScript doesn't currently have a concept of raw bytes. Instead we have to get the bytes from a character representation of the data.

Luckily there's already a standard transfer encoding already heavily in use in various places of the W3C standards... <a href="http://en.wikipedia.org/wiki/Base64">Base64</a>! You can use the <a href="http://tools.ietf.org/html/rfc2397">data:</a> URI scheme to embed image data in your HTML or CSS documents. It's also heavily used for binary data in e-mails.

We can get the data either from an XHR request, from a src attribute or just statically embedded in your JavaScript file. So, now we have our data as Base64 encoded string.

To work with the raw data we need a way to represent bytes. If you're working with ASCII data you can just stick to string representations. But since we're going to be working binary data the most useful way seems to be simple Numbers. That allows you to do bitwise operations and easily convert them to and from ASCII. It's also provides better performance than representing the bytes as Objects.

Now we need a parser. I went with <a href="http://www.codeproject.com/KB/scripting/Javascript_binaryenc.aspx">a sample parser by some guy named notmasteryet</a>. There are others but this seems like a pretty solid implementation and allows us to work with bytes as Numbers. It also works as a reader that lets us read our data piece by piece instead of filling our memory.

<strong>DEFLATE</strong>

The current PNG standard only uses the DEFLATE algorithm for compression. It's the same algorithm used in ZIP, GZIP, zlib, etc. So it's a very common format.

Luckily for us, <a href="http://www.codeproject.com/KB/scripting/Javascript_binaryenc.aspx">notmasteryet's sample</a> also includes a DEFLATE decompressor. It also works as a piece by piece reader which makes it more memory efficient to work with. The reader pattern is a great way to read data in nested formats.

<strong>PNG</strong>

The PNG format consists of a set of named chunks. A set of "IDAT" chunks makes up the main image data. The total data stream is compressed using DEFLATE. The uncompressed data is filtered using one of 5 simple delta compression filters for each line of pixels.

Notice that we haven't yet touched any image-processing specific logic. DEFLATE and delta compression is used for text and other data as much as anything else.

The raw data consists of a color for each pixel. This can be either grayscale, RGB or a reference to a palette color. This is what we really want.

The PNG format is open and <a href="http://www.libpng.org/pub/png/spec/1.1/PNG-Contents.html">well documented</a>. So I'm not going to cover it in any more detail.

<strong>Proof of Concept</strong>

Since we're doing a lightweight JavaScript parser and probably have some control over the image data, we can skip some of the more outlandish features of the specification. We can also skip the verification parts. We'll just skip the file headers and CRC checks.

I decided on an a simple API that reads each line of pixels as an array of RGB colors represented as a number.

{% highlight js %}
var image = new PNG(base64data);
image.width; // Image width in pixels
image.height; // Image height in pixels
var line;
while(line = image.readLine()){
  for(var x = 0;x < line.length;x++){
    var px = line[x]; // Pixel RGB color as a single numeric value
    // white pixel == 0xFFFFFF
  }
}
{% endhighlight %}

I then took that RGB data and inserted the pixels into my document as DIV tags with a background-color.

<p style="text-align: center"><a href="http://labs.calyptus.se/JSBin/Demo/Viewer.html"><img src="http://blog.calyptus.eu/wp-content/uploads/2009/05/gravatar.png" style="display: inline; margin: auto;" alt="Proof of Concept" /></a></p>

In less than 3 hours I had a <a href="http://labs.calyptus.se/JSBin/Demo/Viewer.html">working Proof of Concept</a> of a format I had never worked with before.

<em>I skipped interlacing, alpha and some of the filters for the demo. It's not meant to be a fully working prototype nor a reference library in any way.</em>

<strong>Now What?</strong>

You could...

<ul>
<li>Display the image using a regular rendering method but use the PNG parser to extract colors using a Color Picker.</il>
<li>Add obfuscation or cryptographic layers to render images that can't be easily ripped by bots or downloaded by users.</il>
<li>Render embedded PNG images using VML in Internet Explorer (which lacks data: URI support) with full alpha support.</il>
</ul>

Don't expect this method to become the new hack for PNG or embedded images in Internet Explorer. The rendering methods here are probably too slow for that. You could do some nice stuff with CANVAS though.

However, I have demonstrated that it is possible to work with binary formats in JavaScript. We shouldn't be afraid of utilizing existing binary standards (PNG, GZIP, SVGZ, SWF, TTF...). We shouldn't always fallback to our comfortable old JSON format and reinvent the wheel for every client-side need.

<strong>Relevant Projects</strong>

The <a href="http://www.mootools.net/">MooTools</a> team is working on a tool set for vector graphics in the web browser, <a href="http://github.com/kamicane/art">A.R.T.</a> You could use binary formats to embed your vector based graphics in formats like... TrueType!

The <a href="http://www.ape-project.org/">APE (Ajax Push Engine) project</a> brings socket programming to the JavaScript platform. 

Digg's <a href="http://blog.digg.com/?p=621">MXHR stream</a> parses multipart encoded data and extracts the parts for various uses. This could provide a packaging model for various widgets or data packets.
