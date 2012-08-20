---
layout: page
title: Wind.js文档
---

Wind.js的前身为Jscex，即JavaScript Computation EXpressions的缩写，它为JavaScript语言提供了一个monadic扩展，能够显著提高一些常见场景下的编程体验（例如异步编程）。Wind.js完全使用JavaScript编写，能够在任意支持JavaScript的执行引擎里使用，包括各浏览器及服务器端JavaScript环境（例如[Node.js](http://nodejs.org/)）。

在异步编程的场景下，代码往往会被各种回调函数拆分地支离破碎，因此大家都会构建[各式各样的辅助类库](https://github.com/joyent/node/wiki/modules#wiki-async-flow)来简化开发难度。不过在Wind.js眼中，流程控制本就是语言的职责，只是因为某些限制我们无法使用语言来表达逻辑，于是只能退而求其次地借助于“类库”来解决这方面的问题。开发人员早已熟悉JavaScript中的各式关键字，也已经千百次证明了它们是多么的灵活和简洁，因此在这方面有什么是比JavaScript更为现成的解决方案呢？

Wind.js区别于其他流程控制类库的最大特色，便是纯粹使用JavaScript来表达逻辑，从而避免开发人员学习各种各样缤纷复杂的API。更重要的是，它内置的“运行时动态编译”特性并不会为传统的JavaScript开发带来任何额外的负担。

## Wind.js的特点

* 无需学习额外的API或新的语言，直接使用JavaScript语言本身进行编程。
* 功能强大的异步任务模型，支持并行，取消等常用异步编程模式，经过多种技术平台验证。
* 在任何支持JavaScript语言的环境里直接使用（如浏览器或Node.js），无需额外的编译或转化步骤。
* 基础组件及异步运行库共计4K大小（Minified + GZipped），适合开发网页应用。

## 快速入门

这部分的两个示例将帮助您快速了解Wind.js的基本优势及使用方式，他们分别在浏览器和Node.js环境下执行，您也可以在Wind.js的GitHub中找到这两个示例的完整代码：

* [排序算法动画](./async/samples/browser/sorting-animations/)：在浏览器中绘制动画，演示排序算法的运行过程。
* 博客列表：基于Node.js加载数据，并生成简单的博客列表页面。
* IMDB阅览器：基于HTML 5开发的Windows 8应用程序。

## 模块

Wind.js以模块化形式分发，目前主要有以下几个模块：

* [核心模块](./core/)（TBD）
* [编译器模块](./compiler/)（TBD）
* [构造器基础模块](./builderbase)（TBD）
* [异步模块](./async/)
* [Promise模块](./promise/)（TBD）

关于模块引入方式，请阅读各模块的文档以及[加载模块](./importing/)相关内容。

## 反馈

假如您在Wind.js使用过程中产生任何疑问或是意见建议，请加入[Wind.js讨论组](https://groups.google.com/d/forum/windjs)，或直接[发送邮件](mailto:windjs@googlegroups.com)提问或参与讨论。对于使用过程中发现的Bug，也可以[汇报至GitHub](https://github.com/JeffreyZhao/wind/issues)。