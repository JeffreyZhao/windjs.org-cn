---
layout: post
title: "无法封装的函数：eval"
date: 2012-09-02 18:37
categories: eval
---

从Wind.js诞生之日起就依赖`eval`实现了运行时的代码转化，但也不断引起许多人对Wind.js的误解。许多人一看到`eval`便对Wind.js敬而远之，但可能他们一直在使用的组件或是应用便大量运用了`eval`。还有人抱怨Wind.js的方法定义方式太冗余，假如`eval(Wind.compile(...))`可以进一步简化即可。其实这两个问题的解决方法都是一种：将`eval`封装起来，只在类库内部使用。只可惜我们无法做到这一点。

<!-- more -->

`eval`是个特殊的函数，它的作用是动态执行代码。JavaScript的代码在执行时不是一个人在战斗，而是要结合上下文才能得到真正的执行结果。例如同样一句`eval("i")`，在不同的上下文中将会得到不同的结果：

    (function () {

        var i = 0;
        console.log(eval("i")); // 0

    })();
    
    (function () {

        var i = 1;
        console.log(eval("i")); // 1

    })();

也正因为如此，在Wind.js中我们无法封装`eval`函数，因为我们无法避免一个Wind.js方法使用外部的上下文，例如：

    (function () {

        var hello = "world";
    
        var printHello = eval(Wind.compile("async", function () {
            console.log(hello);
        }));
    
    })();

请注意这个`hello`变量处于方法外部，我们期望它的效果等价于：

    var hello = "world";

    var printHello = function () {
       ...
       console.log(hello);
       ...
    }

这也是现在的`eval`的效果，但是假如我们将eval封装至Wind.compile内部，则会近似成为：

    Wind.compile = function () {
        var code = "(function () {" +
            /* ... */
            "console.log(hello);" +
            /* ... */
        "})"
        
        return eval(code);
    }
此时字符串中函数则会使用`Wind.compile`内部当前的上下文，自然也已经找不到`hello`，即便有也很可能不是我们所需要的那个。
如果您对于`eval`为何无法封装有所疑惑的话，不妨[执行以下代码](http://jsfiddle.net/jeffz/Gdyb2/)便会一目了然：
    (function () {

        var getCode = function (f) {
            return "(" + f.toString() + ")";
        }
        
        var getEvaledCode = function (f) {
            return eval(getCode(f));
        };
        
        var test1 = function () {
            var i = 0;
            var f = eval(getCode(function () {
                console.log("i = " + i);
            }));
            
            f();
        };
        
        var test2 = function () {
            var i = 0;
            var f = getEvaledCode(function () {
                console.log("i = " + i);
            });
            
            f();
        };
        
        // 输出“i = 0”
        test1();
        
        // 输出“i is not defined”
        try {
            test2();
        } catch (ex) {
            console.log(ex);
        }

    })();

换句话说，只要Wind.js方法内部还会继续使用上下文的成员，则无可避免需要我们在代码里使用`eval`。不过换句话说，只要能够保证方法内部不会用到上下文，则`eval`函数的调用就可以被隐藏起来，甚至可以使用`new Function()`来代替。只可惜Wind.js不能给使用者做出这样的限制，但如果有其他组件可以提供这样的约束，便可以让用户摆脱对`eval`的直接依赖。

例如[SeaJS](http://seajs.org/docs/)。因为在SeaJS中，每个模块都可以认为是独立存在，而不需要依赖上下文的单元。