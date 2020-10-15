---
layout: post
title: 使用VueJS脚本gadgets规避防御
---

## 介紹
我们发现，流行的JavaScript框架VueJS提供了对网站安全有严重影响的功能。如果您遇到使用Vue的Web应用程序，本篇文章将帮助您识别由gadgets脚本创建的Vue特定XSS向量，您可以使用这些gadgets脚本来利用目标。

gadgets脚本是由框架创建的额外功能，可以导致JavaScript执行。这些功能可以是JavaScript或基于HTML的。gadgets脚本通常对绕过WAF和CSP等防御措施很有用。从开发者的角度来看，了解一个框架或库创建的所有gadgets脚本也是很有用的；当允许用户在自己的Web应用程序中输入时，这些知识可以帮助防止XSS漏洞。在本篇文章中，我们将涵盖从基于表达式的向量到突变XSS（mXSS）等多种技术。

### 注意
这里有很多信息!如果你对学习攻击感兴趣，你可能会想读完这本书。但如果你遇到了特定的场景，只是需要一个解决它的向量，你可以直接跳到我们[XSS Cheat Sheet](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet#vuejs-reflected)中新更新的VueJS部分。

在这篇文章中，我们将涵盖:

指令
缩短有效载荷
事件
突变
为V3调整有效载荷
传送
使用案例


## 一切都是从哪里开始的？
在[推特](https://twitter.com/PortSwiggerRes/status/1265647826383634432?s=20)上讨论各种 VueJS 攻击的时候，我、[Lewis Ardern](https://twitter.com/LewisArdern) 和 [PwnFunction](https://twitter.com/PwnFunction) 决定创建一篇博文来更详细地介绍它们。我们的合作非常有趣，并想出了一些有趣的向量。这一切都始于尝试减少以下[VueJS XSS向量](https://portswigger-labs.net/xss/vuejs2.php?x=%7B%7BtoString().constructor.constructor(%27alert(1)%27)()%7D%7D)。
```
{{toString().constructor.constructor('alert(1)')()}}
```
-----

Poole is the butler for [Jekyll](http://jekyllrb.com), the static site generator. It's designed and developed by [@mdo](https://twitter.com/mdo) to provide a clear and concise foundational setup for any Jekyll site. It does so by furnishing a full vanilla Jekyll install with example layouts, pages, posts, and styles.

This demo site was last updated {{ site.time | date: "%B %d, %Y" }}.

There are currently two themes built on Poole:

* [Hyde](http://hyde.getpoole.com)
* [Lanyon](http://lanyon.getpoole.com)

Learn more and contribute on [GitHub]({{ site.github.repo }}).

## What's included

Poole is a streamlined Jekyll site designed and built as a foundation for building more meaningful themes. Poole, and every theme built on it like this one, includes the following:

* Complete Jekyll setup included (layouts, config, [404]({{ '404.html' | relative_url }}), [RSS feed]({{ 'atom.xml' | relative_url }}), posts, [archive page]({{ 'archive' | relative_url }}), and [example page]({{ 'about' | relative_url }}))
* Mobile friendly design and development
* Easily scalable text and component sizing with `rem` units in the CSS
* Support for a wide gamut of HTML elements
* Related posts (time-based, because Jekyll) below each post
* Syntax highlighting, courtesy Jekyll's built-in support for Rouge

Additional features are available in individual themes.

## Browser support

Poole and its themes are by preference a forward-thinking project. In addition to the latest versions of Chrome, Safari (mobile and desktop), Firefox, and Edge.

## Download

These themes are developed on and hosted with GitHub. Head to the [GitHub repository]({{ site.github.repo }}) for downloads, bug reports, and features requests.

Thanks!
