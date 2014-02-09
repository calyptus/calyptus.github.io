---
layout: post
title: Choosing JavaScript Module Dependency Syntax
date: 2011-10-08 14:39:32.000000000 +02:00
categories:
  - seb
tags:
  - javascript
  - module-dependancy
---
*TLDR This post is mainly a way for me to structure arguments and counter-arguments for various JavaScript module systems. You can jump to the bottom of the page for a matrix mapping features to various module systems. I'll keep this page updated as new arguments are accepted or new module systems are invented, if they add new benefits.*

*I make the assumption that you should always compile scripts together and gzip in production (possibly to multiple files). I make a second assumption that the complexity of such a build tool is irrelevant once it’s already developed. If you don't accept these assumptions, the arguments below might be irrelevant or incomplete.*

## Introduction
JavaScript originally didn't specify a way to work with source code in multiple different files, or across different modules in the same file. Recently we have seen many different ways to solve this approach mainstream JavaScript development. I'll try to break them down and explain the reasoning for choosing between them in today's JavaScript environments.

There are three primary attributes that a module system might include:

**Module Isolation** - Code within a module should not leak in to the global scope shared by multiple modules.

**Inter-module Dependency Definition** - There should be some syntax for defining dependencies between multiple source code files.

**Script Loader** - Some how the application code from all relevant files must be loaded at runtime. The module system might use the same technique during development and release, but often you should treat them differently. In a production environment, your focus is on fast loading. In debug, your focus is on ease of use and mapping VM breakpoints and errors to readable source code.

**Sandboxing and Inversion of Control** - This is currently a rarely used feature. It involves a way to load a module dependency graph in a different context or sandbox. It is an important part of dependency loading. It's beyond the scope of this post, but it can be implemented together with any custom script loader.

## The Traditional JavaScript Library
Traditionally JavaScript libraries handled module isolation by wrapping source code in a (function(){ ... })() closure.

Inter-module Dependencies didn't have a specific syntax. One module gained access to another through a namespace. A namespace was defined in the global scope. Modules could collide if the same namespace was used by two different modules.

On the web, loading of dependent files was managed by defining all application dependencies in a long list using &lt;script&gt; tags.

Note: If one &lt;script&gt; tag fails to load, others may still run. That means this model is fragile in a production environment. If one of your dependencies isn't loaded, other code may still run. It might not be noticeable at first. You can see this with Chrome sometime. It uses various "smart" network techniques that sometimes fail to load a script dependency on a page, causing some features to break, sometimes in very bad ways.

In production, the best practice is to compile all these code files into a single minified and compressed JavaScript file. This ensures that all dependencies are loaded together, or not at all. It also makes sure the script is small by minifying the source. Putting it all into a single file makes good uses of GZIP compression and ensures minimal overhead from the transfer protocols.

The problem with the traditional approach is that there isn't a specified way to define dependencies between various script files. Therefore it's difficult to know what scripts need to be included to run a program and in what order they need to run. Together they form a dependency graph.

## Externally Defined Dependencies
The module isolation is done as in traditional JavaScript libraries. The creation of the dependency graph and script loader is a little different though.

We can define the dependency graph in an external file. E.g. we can include a package file which defines references to all the files in a packages and how they relate to each other.
{% highlight js %}
{
  '/MyDependentModule.js': ['/lib/StandAloneModule.js'],
  '/lib/StandAloneModule.js': []
}
{% endhighlight %}
A custom script loader would first load a package file, then load and execute all scripts in the order they need to be executed. We can also add some asynchronous loading magic. This is the most efficient way to load scripts and easy to do.

It is easier to maintain than list of scripts because we don't have to worry about the ordering of the scripts. The loader calculates that. On the down side, it's still rather painful to maintain since you have a separate file that needs to be maintained when dependencies change.

## Inferred Dependencies
You can try to infer the dependency graph by parsing the code and infer what global namespaces are defined in one file and what namespaces are used by another.

{% highlight js %}
var MyGlobalModule = (function(){
  var imported = MyOtherModule;
  ...
  return exports;
})();
{% endhighlight %}

Since we're not in a "with" statement we can infer the MyOtherModule must be in the global scope. Since we're not defining it, someone else must be. Therefore we have a dependency on what ever file defines MyOtherModule.

Modules that don't define a new object but rather extend existing ones (monkey patching) doesn't necessarily have an exports. You can easily add an empty object to create a dependency on these pseudo-modules.

One problem with this approach is that we need to have a list of all files available. Then pre-parse them so that we can see which files are used and in what order they need to be executed. This is easy enough for a local compiler. It can just scan all files in a directory. An in-browser script loader for use during development can't list files though.

One way to solve this is to have a convention where the name of a global variable maps to the name of a file. That way we can also infer the name and location of the file.

This technique hasn't gained much traction. Possibly because it takes some effort to make the tools. Once the tooling is done, that's not an argument not to use it.

## Comment Defined Dependencies
Instead of inferring our dependencies using a convention, we define them in a comment within the source file.

{% highlight js %}
// @requires /lib/MyOtherModule.js
var MyGlobalModule = (function(){
	var imported = MyOtherModule;
	...
	return export;
})();
{% endhighlight %}
The script loader can load the script file using XHR. Then parse the file's comments to see what dependencies it has. Then load the dependency using XHR etc. After the dependency graph has been loaded this way, they can be executed in order using &lt;script&gt; tags.

This has the added benefit that the file names doesn't necessarily need to correspond to global object names.

This is also easy to implement which is why it has gained more traction than inferred dependencies.

## Synchronous Module Definition (CommonJS)
The CommonJS wiki originally defined a synchronous API to load dependencies in non-browser environments. Module isolation between files is built-in. So, the global scope isn't shared. Only exported properties are shared.

Inter-module dependencies are defined using a synchronous API. You call the global require function passing a string argument representing the name of another source file.
{% highlight js %}
var imported = require('lib/MyOtherModule');
...
exports.myExport = ...;
{% endhighlight %}
These modules can be used as is, in Node.js and other CommonJS compatible implementations. (Much fewer than JavaScript compatible environments since CommonJS is not a formal standard.)

This model can be used in a browser environment using a custom script loader. The script loader can use XHR to load the script, parse the require statements, load the next script, parse it and build the dependency graph.

To actually execute modules in isolation, you need to wrap them in some custom closure. Then the code is executed by inject the new code into script tags.

## CommonJS Hybrids
To make a script compatible with both CommonJS and the traditional model you can use a hybrid model. Your source file checks for existence of CommonJS require/exports and uses them if available, otherwise it falls back on the traditional model. This makes source code directly compatible with CommonJS and the traditional model out-of-the-box.

## Define Block
{% highlight js %}
define(function(require, exports, module){
	var imported = require('lib/MyOtherModule');
	...
	exports.myExport = ...;
});
{% endhighlight %}

The define() statement that enables async running. That's half way to AMD. This solves some debugging issues in current browsers. However, it doesn't support defining the dependency list before running the code. Therefore the loaders still won't work automatically in browser environments.

**UPDATE: This was supported in Node.js for a while but is no longer supported in 0.5.x.**

## Asynchronous Module Definition (AMD)
Asynchronous Module Definition combines module-isolation, dependency definitions and script-loading into one tool, to make it easier to use the code in the browser.

{% highlight js %}
define(['MyOtherModule'], function(imported){
...
return exports;
});
{% endhighlight %}

This requires a custom loader. The custom loader is efficient than the custom loaders used by Infered Dependencies, Comment Defined Dependencies and Synchronous Module Definitions. It can also work with Cross-Site Scripting (XSS) dependencies without the Cross-Origin Resource Sharing protocol (CORS). However, the Externally Defined dependency graph is even more efficient.

It is possible to create and AMD/CommonJS/Traditional hybrid that supports all formats directly from source. This requires a lot of boilerplate code and is not easy to maintain.

## ECMAScript.Next Modules
The ECMAScript Harmony proposals include a new syntax for defining modules. This will probably be in ECMAScript.Next (probably ECMAScript 6).

All module scope will be isolated by default. Dependencies are defined using a new syntax. The new syntax ensures that module dependencies can be statically inferred (without relying on convention).

{% highlight js %}
module importedModule from "MyOtherModule.js";
...
export myExport;
{% endhighlight %}

Just like Synchronous Module Definitions, this requires code rewriting. The custom script loader will execute the rewritten code after all dependencies has been loaded.

This syntax has the added advantage of all the other ECMAScript.Next syntax that you can use as well.

## Static Compilation
All systems except (A)synchronous Module Definitions and Infered Dependencies allow dynamic naming of modules based on some programmatic logic. I.e. you can call the require function with a variable require(selectedModule + '/subComponent'); Don't do this. If you do, you can't statically infer the dependencies and you can't compile your script together without executing it. I'm going to assume the convention to always define a constant string.

Regardless of what syntax you use for defining dependencies, you can use a compiler and minifier to package them up. There are some benefits of using a custom script loader even in production because you can load scripts in parallell and from several servers. The best way is to use an Externally Defined Dependency Graph. That way, you can start loading files without first loading the files that dependent upon them.

Asynchronous Module Definition (AMD) by definition includes a fairly efficient script loader for the browser. You may be tempted to use it as-is in production. However, you should package related files together and at least minify them (as stated above). Therefore you're already introducing a compilation step. That compilation step can be used to turn any other syntax into AMD or better yet, generate an Externally Defined Dependency Graph.

Because you should **never** put raw source code in a production web app, all syntaxes are equals in terms of production browser script loading. However, during development you may well want to load source code without a compilation step and different syntaxes can provide some advantages there.

## Choosing Module Dependency Syntax
<table class="compatibility" style="margin: auto;" border="0" cellspacing="1" cellpadding="0">
	<tbody>
		<tr class="top-head">
			<td></td>
			<th><span>Traditional</span></th>
			<th><span>External</span></th>
			<th><span>Comment</span></th>
			<th><span>Inferred</span></th>
			<th><span>CommonJS</span></th>
			<th><span>Hybrid</span></th>
			<th><span>Define Block</span></th>
			<th><span>AMD</span></th>
			<th><span>AMD Hybrid</span></th>
			<th><span>ES.Next</span></th>
		</tr>
		<tr>
			<th><span>Easily Maintained / Clean Syntax</span></th>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: yellow;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: red;"></td>
			<td style="background: lime;"></td>
		</tr>
		<tr>
			<th><span>Source Compatible with Traditional Model</span></th>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: red;"></td>
			<td style="background: lime;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: lime;"></td>
			<td style="background: red;"></td>
		</tr>
		<tr>
			<th><span>Source Compatible with Node.js (Out-of-the-box)</span></th>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: lime;"></td>
			<td style="background: red;"></td>
		</tr>
		<tr>
			<th><span>Analysis/Completion Tooling in Current IDEs</span></th>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: red;"></td>
			<td style="background: yellow;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: yellow;"></td>
			<td style="background: red;"></td>
		</tr>
		<tr>
			<th><span>Debugging in WebKit</span></th>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: red;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: red;"></td>
		</tr>
		<tr>
			<th><span>Debugging in Other Browsers</span></th>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: yellow;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: yellow;"></td>
		</tr>
		<tr>
			<th><span>XSS Development Loader using CORS</span></th>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
		</tr>
		<tr>
			<th><span>XSS Development Loader without CORS</span></th>
			<td style="background: lime;"></td>
			<td style="background: yellow;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: red;"></td>
		</tr>
		<tr>
			<th><span>file:// Protocol Development Loader</span></th>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: red;"></td>
		</tr>
		<tr>
			<th><span>Fast Development Loader (Large Code Base, Cold Start)</span></th>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: yellow;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: red;"></td>
		</tr>
		<tr>
			<th><span>Fast Production Loader</span></th>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
			<td style="background: lime;"></td>
		</tr>
		<tr>
			<th><span>ES.Next Features (Classes, Destructuring etc.)</span></th>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: red;"></td>
			<td style="background: lime;"></td>
		</tr>
	</tbody>
</table>

For me, maintainability and less boilerplate is important. Therefore the Traditional, External and AMD Hybrid model are not acceptable.

The main reason to choose the AMD syntax is if you have a very large code base. For those cases, pre-parsing may be slow during development. You can also cache pre-parsed dependencies so that only changed files are updated. This means the others can be fast the second time you run a large code base in development.

Another case to use AMD is if you need cross-site scripting dependency support in IE6/7 or against a third-party server without CORS support, during development. This is a much more esoteric use-case. I've had the need for it, but not during development. Only in production, and then we're only using compiled code.

If you do need source compatibility with the traditional model and/or CommonJS, then choose either Inferred, CommonJS or the Hybrid model. This doesn't require your users to download a separate bootstrapper to run your source.

If you accept that you require a bootstrapper. Then you can use the ES.Next model which gives you the added benefit of many new syntax features. There is a problem debugging code that is rewritten on-the-fly in some WebKit Inspector implementations. This is a passing problem and we'll soon have full symbol mapped debugging for new source languages.

In fact, all the problems with the ES.Next model are likely a passing problems while the other's remain. It's going to be the model to replace them all in the long term.

If you have more module syntaxes or more arguments for one model over another, please drop a comment. A lot of times we base our decisions on intuition rather than reasoned arguments. I've tried to break it down without intuitive arguments (like "it feels cleaner or feels simpler").
