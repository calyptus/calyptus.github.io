---
layout: post
title: Client Side Dependency Strategy
date: 2009-05-17 14:06:00.000000000 +02:00
categories:
  - seb
tags:
  - javascript
  - clientside
---
#### This post is in response to an off-site discussion about modular dependency strategies. But I figured I'd post it here for future reference.

The <a href="http://blog.calyptus.eu/source-code/">Calyptus Web Resource Manager</a> is a project that can on compile-time or on runtime handle your JavaScript, CSS, and other client-side dependencies. You can keep source code as separate files on the server or pre-compile packages (such as .ZIP, .DLL or .JAR). Currently source is available only on the .NET/Mono platforms but the concept is valid for all platforms. 

## Syntax
The syntax is largely inspired to be compatible with <a href="http://www.scriptdoc.org/">ScriptDoc</a> and <a href="http://www.ecmascript.org/es4/spec/overview.pdf">ECMAScript 4 Draft</a> import statement. In the top of your file you add the dependencies that your file relies on:

{% highlight js %}
/*
@import [package, ]filename
@include [package, ]filename
@build [package, ]filename
@compress [always|release|never]
*/
{% endhighlight %}

*Don't worry, we're not going to ruin your precious open-source project with inline docs. Read on.*

**@import** - Indicates that this file has a dependency on the referenced file and that it needs to be included in the final document (implicitly before this one). The other file may be a JavaScript file, CSS, image, Flash or something else. The project is fully extensible.

**@include** - Same as @import but also indicates that the referenced file should be merged into this one on compile or runtime.

**@build** - Same as @include but also merges any nested @import statements. Allowing you to create a single packaged file.

**@compress** - Indicate whether the document should use a compression tool (such as YUI compressor) or not. Defaults to "release", which means that it won't compress during the debug stage.

You can reference a file by either filename/namespace or package + filename/namespace. You may include wild cards to reference an entire path or namespace. If you're referencing another file in the same package, you can exclude the package name.

If you are running ASP.NET you can exclude the package if you're referencing an assembly that is already referenced in your Web.config.

If you're in a .js file, the filename will automatically look for files ending in .js.

This allows you to do a namespace like syntax on prepackaged files:

{% highlight js %}
/*
@import MooTools.Core.*
@import MooTools.More.URI
*/
{% endhighlight %}

If you want to use the runtime view generating tools the syntax depends on what View Engine you're running. For ASP.NET WebForms you can use the following controls:

{% highlight xml %}
<c:Import src="filename" runat="server" />
<c:Import assembly="package" name="namespace/filename" runat="server" />
<c:Include ... />
<c:Build ... />
{% endhighlight %}

In the future this will be integrated into the ASP.NET ScriptManager as well. For other view engines the syntax would be much prettier.

## Example
*MyBaseStyle.css*
{% highlight css %}
div.BaseClassItem {
  background-image: url(MyBaseImage.png);
}
{% endhighlight %}

*MyBaseClass.js*
{% highlight js %}
// @import MyTheme.css
var MyBaseClass = new Class({
  initialize: function(){
    this.element = new Element('div', { className: 'BaseClassItem' });
  }
});
{% endhighlight %}

*MyChildClass.js*
{% highlight js %}
// @import MyBaseClass.js
var MyChildClass = new Class({
  Extends: MyBaseClass,
  ...
});
{% endhighlight %}

*MyView.aspx*
{% highlight xml %}
<c:Include src="MyChildClass.js" />
{% endhighlight %}

*OUTPUT:*
{% highlight xml %}
<link href="MyBaseStyle.css" rel="stylesheet" type="text/css" />
<script src="MyBaseClass.js" type="text/javascript"></script>
<script type="text/javascript">
var MyChildClass=new Class({Extends:MyBaseClass,...});
</script>
{% endhighlight %}

Since I used the included command the file is included in the output document. All it's dependencies are automatically added to the document through links.

Any referenced file is only added once to the output. So it's no problem adding multiple references to the same resource in partial views or by indirect dependencies.

*MyOtherView.aspx*
{% highlight xml %}
<c:Import src="MyChildClass.js" />
<c:Import src="MyBaseClass.js" />
{% endhighlight %}

*OUTPUT:*
{% highlight xml %}
<link href="MyBaseStyle.css" rel="stylesheet" type="text/css" />
<script src="MyBaseClass.js" type="text/javascript"></script>
<script src="MyChildClass.js" type="text/javascript"></script>
{% endhighlight %}

In the sample above, I import the base class after the child class. Since the child is dependent on the base, it will be included first. Therefore the second reference to MyBaseClass.js is excluded.

## Typical Work Flow - Late Optimization
Typically you would only use the @import statement in all your resources. You should only reference any direct resources that your code or style sheet uses. Indirect files are referenced by the referenced resources so that if a dependency changes, you don't have to update all your reliers. Your views will only reference the direct resources that it is using by import statements as well.

This will generate a lot of &lt;script&gt; and &lt;link&gt; tags in your documents. This is not good for production where you want to minimize the overhead of multiple requests. That's when you start building clusters.

*Common.css*
{% highlight css %}
/*
@build Headers.css
@build Footers.css
@build MyBaseStyle.css
*/
{% endhighlight %}

*Common.js*
{% highlight js %}
/*
@build MooTools.Core.Fx.Tween
@build MyChildClass.js
*/
{% endhighlight %}

Now I can include the cluster Common.js in my view:

{% highlight xml %}
<c:Import src="Common.css" />
<c:Import src="Common.js" />
...
<c:Include src="MyChildClass.js" />
{% endhighlight %}

*OUTPUT:*
{% highlight xml %}
<link href="Common.css" rel="stylesheet" type="text/css" />
<script src="Common.js" type="text/javascript"></script>
{% endhighlight %}

The MyChildClass.js reference and all it's dependencies are ignored since those file has already been included in the document by Common.css and Common.js. You can for example add these clusters to your Master view to automatically optimize all your partial views. If you remove a reference from your cluster it won't break any of your code, since those files are individually added by your partial views to your document.

This pattern will allow you to do late optimization of your load-time by grouping only the files that are commonly used in to clusters. Leaving edge-case files into the outer branches of your site. To accomplish this I recommend that you use a modular framework such as <a href="http://www.mootools.net/">MooTools</a>.

Your clusters should be named and composed in relevant packages for your site, not in packages of JavaScript frameworks. For example, DON'T create a MooTools.js cluster that includes all MooTools files.

*By default, @include and @build commands are evaluated as @import during the debug stage. That makes it easy to find the references to your source code with debugging tools such as FireBug.*

## Messing Up Your Beautiful Source? Use Place Holders
If you're working with a consultant project you can just put all your references in the source file. That makes it very easy to work with. But if you have an open-source project you may not want to mess up the source with dependency references. Instead, use place holder files that @include the original source and references the dependency place holders using @import.

*Fx.js*
{% highlight js %}
/*
@import Class.Extras.js
@include Real/Source/Fx/Fx.js
*/
{% endhighlight %}

*Fx.CSS.js*
{% highlight js %}
/*
@import Fx.js
@import Element.Style.js
@include Real/Source/Fx/Fx.CSS.js
*/
{% endhighlight %}

Now you can reference your place holders to get dependencies instead of the original source files.

## What about my CDN?
You can use a CDN to store your clusters. Just reference the full URIs in your import statements. There is a pre-built class that does this with MooTools on Google. Just @import GoogleAPIs.MooTools.

I will add an **@embedded** syntax to reference other files that have already been included. That way you could write your own like this:

*MooTools-Cluster-Google.js*
{% highlight js %}
/*
@import http://ajax.googleapis.com/ajax/libs/mootools/1.2.2/mootools-yui-compressed.js
@embedded MooTools.Core.*
*/
{% endhighlight %}

If you reference this cluster in your view, all references to your local MooTools files will be ignored since it they are already included in the Google cluster.

## @include on Images
If @include filename.png is used in a style-sheet, every instance of url(filename.png) will automatically be replaced with base64 embedded data at runtime. This is only used on the runtime version since this content can't be sent to IE browsers. IE browsers will get the url(filename.png) reference intact.

This also works with view/document Include commands. In that case an &lt;img&gt; tag is rendered with a link or embedded content depending on the browser capabilities.

This pattern allows you to do late load time optimization of image dependencies.

## Getting Started
As always, begin by <a href="http://blog.calyptus.eu/source-code/">checking out the source</a>.
