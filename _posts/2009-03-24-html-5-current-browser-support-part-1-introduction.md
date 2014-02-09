---
layout: post
title: HTML 5 Current Browser Support - Part 1 - Introduction
date: 2009-03-24 00:02:13.000000000 +01:00
categories:
  - seb
tags:
  - compatibility
  - dom
  - html-5
  - javascript
  - mootools
  - w3c
  - whatwg
---
The <a href="http://www.whatwg.org/specs/web-apps/current-work/">HTML 5 working draft</a> is continuing it's development of the future support for HTML 5. This includes new tags, attributes and a strong specification of how clients should interact with old and new elements. What I find even more intriguing, is the standardization of many advanced JavaScript DOM features (such as editable content, drag and drop). Most of which has been available to IE users for more than a decade. This is one area that standards has been particularly slow to adopt. With the current beta versions of Safari, Chrome and Firefox these new browsers are finally ready to leave IE behind (yes, even IE 8).

Many people are still frightened of implementing code according to a working draft. Especially since it's <a href="http://wiki.whatwg.org/wiki/FAQ#When_will_HTML_5_be_finished.3F">not scheduled to be complete until 2012</a>. In my opinion, those fears are largely unfounded at this point. The primary reason for this is that many of the features have been available in IE for many years and the <a href="http://www.whatwg.org/specs/web-apps/current-work/">HTML 5 specification</a> centers around keeping some historical compliance. So the primary threat for lagging cross browser functionality has already been eliminated. It is also the <a href="http://www.whatwg.org/">WHATWG</a>'s estimate that browsers will have full compliance and people will have started utilizing this new standard long before it is finalized. For these reasons, by the time you read this, you may already be a late adopter.

However, there are still some quirks that you need to be aware of. I've been working on cross browser layers of the HTML 5 specifications since 2007 including backwards compatible code for older browsers. This code has been used in production and little of it has changed since mid-2008. Therefore I've <a href="http://github.com/calyptus">started work</a> on introducing these features to my JavaScript framework of choice, <a href="http://www.mootools.net/">MooTools</a>. While I refactor my code for this purpose I thought I might introduce some of the quirks that you might come across in your own endeavors.

##Coming up
 * Part 2 - Drag and Drop, Copy and Paste
 * Part 3 - Range and Selection
 * Part 4 - ContentEditable and ExecCommand
