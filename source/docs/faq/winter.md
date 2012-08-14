---
layout: page
author: winter
title: Wind.js FAQ
---

## Wind.js是做什么的？

在使用JavaScript之时，一旦涉及http请求、等待时间、IO操作等场景，往往需要使用异步操作。而受JavaScript语言之限，同步写法和异步写法往往差距甚大。请看以下代码：

假若要编写一个函数，向页面中插入10个div元素，写法如下：

    var insert = function () {
        for(var i = 0; i < 10; i++) {
            var div = document.createElement("div");
            document.body.appendChild(div);
        }
    }

假若延迟插入元素的操作，每一秒插入一个元素，则写法如下：

    var insertEverySecond = function () {
        var i = 0;
        var div = document.createElement("div");

        setTimeout(function () {
            var div = document.createElement("div");
            document.body.appendChild(div);    
            if(i++ < 10)
                setTimeout(arguments.callee, 1000);
        }, 1000);
    }

二者之间逻辑非常相似，但是JavaScript语言的限制导致了它们代码结构有很大差异。而Wind.js则是一个用于解决代码书写结构问题的类库。使用Wind.js实现代码对比如下：

    var insert = function () {
        for (var i = 0; i < 10; i++) {
            var div = document.createElement("div");
            document.body.appendChild(div);
        }
    }

    var insertEverySecondByWind = eval(Wind.compile("async", function () {
        for (var i = 0; i < 10; i++) {
            $await(Wind.Async.sleep(1000));
            var div = document.createElement("div");
            document.body.appendChild(div);
        }
    }));
    
可见除了定义方法时多了一部分“框架代码”，代码整体逻辑结构与之前可谓如出一辙，而“暂停”功能即可通过Wind.js异步模块中的`$await`操作配合`sleep`方法来实现。

## 如何搭建Wind.js运行环境？

如果你在使用支持npm的环境（如Node.js），可以直接安装npm包：

    npm install wind

然后通过require来使用：

    var Wind = require("wind");

如果你在使用浏览器环境，则可以引用wind.js的文件：

    <script src="wind-core-x.y.z.js"></script>
    <script src="wind-compiler-x.y.z.js"></script>
    <script src="wind-builderbase-x.y.z.js"></script>
    <script src="wind-async-x.y.z.js"></script>

这里, x.y.z为版本号。

## Wind.js与其它异步类库相比有什么不同？

简单地说，Wind.js认为：

* 解决异步问题不应该像[它们](https://github.com/joyent/node/wiki/modules#wiki-async-flow)一样对使用者加以限制。
* 开发者仅仅需要学习“JavaScript”就够了，不应该让开发者额外的API。
* 应当让开发者可以使用任何JavaScript的调试环境来开发，不应当改变开发者的工作习惯。
* 类库应该着重于解决核心问题，不应当因为堆砌功能而使自己的体积膨胀（事实上经过gzip后，Wind.js核心和异步组件总共仅4k大小）。

## Wind.js为什么使用eval，难道没听说过eval is evil吗？

第一，很遗憾，对于Wind.js而言eval是必要的，而且没有任何已知办法将eval封装起来，因为Wind.js必须要使用者显式调用eval来保证当前的作用域可访问。

第二，当使用AOT编译方式发布时，发布的代码中是没有eval的，eval这样的写法可以保证你的代码无需经过编译（或其他“预处理”方式）即可在任何JavaScript环境中调试。

第三，eval is evil实际上是个误区，任何语言特性都有其存在的意义和适用之处。Wind.js中的eval运行次数跟代码规模线性相关，这种程度的使用对于性能影响微乎其微。