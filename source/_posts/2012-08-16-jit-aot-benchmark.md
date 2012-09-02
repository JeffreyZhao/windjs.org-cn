---
layout: post
title: "代码预编译前后性能对比"
date: 2012-08-16 22:28
categories: eval 性能 预编译
---

Wind.js在推广过程中遇到的最大的问题之一便是众人对`eval`的反对，许多人见到Wind.js代码中需要显示调用`eval`便心生惧意，敬而远之。其中对于`eval`最大的质疑便是其性能问题，尽管Wind.js提供了[预编译器]({{root_url}}/docs/aot/)，但还是有个别专家强烈反对Wind.js中的`eval`使用方式，认为它会严重降低程序性能，以至于在“开发模式”中都不该出现`eval`。

为此我进行了[一次简单的性能对比](http://blog.zhaojie.me/2012/08/js-code-from-eval-benchmark.html)，从结果上看似乎由`eval`生成的代码与直接定义的代码在性能上几乎完全相同，但是[也有证据显示](http://www.otakustay.com/eval-performance-profile/)，`eval`会对代码整体性能产生负面影响，甚至只需要在代码中出现`eval`而根本无需执行。由于JavaScript引擎的实现各不相同，优化手段也十分复杂，甚至是一个黑盒（即便开源也很难分析完整），因此要完整得到`eval`等特定功能对于性能的影响是件难度极大的事情，甚至由于变化太多，即便得出结论其参考价值也十分有限。为此，我还是打算以性能比较的方式来尝试说明“特定条件”下的情况。

这次的性能对比，便是要对比一份使用了Wind.js的代码，在预编译前后的性能差距。由于`eval`在Wind.js中的使用方式十分单一，毫无变化，因此这次实验对于实际情况下的Wind.js使用还是具有一定参考价值的。

<!-- more -->

## 实验目标

在之前的实验中，有人认为比较用例设计地不够具有代表性，主要是指三个方面：

* 代码过于简单，仅是简单的计算，不够具有代表性。
* 整段代码处于`global`范围，而不是闭包内部。
* `eval`中的代码没有访问外部成员。

关于第一点，其实设计“具有代表性”的用例实在不是一件容易的事情，因为在我看来Wind.js本身完全不会用来进行密集的性能计算，所有被`eval`的方法都是包含大量异步操作和流程逻辑的代码，因此我始终不太做一些所谓的“性能测试”，怕给用户带来错误的引导。

不过后两点说的的确有一定道理，由于缺乏默认的封装特性，主流的JavaScript标准写法都会将代码存放在一个大的闭包内，以免污染到全局对象，例如：

    (function () {
    
        /* 实际代码 */
        
    })()
    
而在方法内部访问外部上下文的成员更是一件再普通不过的事情了。

## 实验代码

为此，我重新设计了一个性能比较的实验，至少可以体现Wind.js比较常见的使用场景。这里我选择的示例是[快速排序](http://en.wikipedia.org/wiki/Quicksort)。这里我并不打算讲解快速排序的实现方式，而是仅仅给出实验代码：

    var sort = (function () {

        var numbers = []

        var array;

        var compare = function (x, y) {
            return x - y;
        };

        var swap = function (i, j) {
            var t = array[i];
            array[i] = array[j];
            array[j] = t;
        };

        var quickSort = eval(Wind.compile("async", function () {

            var partition = function (begin, end) {
                var i = begin;
                var j = end;
                var pivot = array[Math.floor((begin + end) / 2)];

                while (i <= j) {
                    while (true) {
                        if (compare(array[i], pivot) < 0) { i++; } else { break; }
                    }

                    while (true) {
                        if (compare(array[j], pivot) > 0) { j--; } else { break; }
                    }

                    if (i <= j) {
                        swap(i, j);
                        i++;
                        j--;
                    }
                }
                
                return i;
            };
            
            var sort = function (begin, end) {
                var index = partition(begin, end);

                if (begin < index - 1) 
                    sort(begin, index - 1);

                if (index < end) 
                    sort(index, end);
            };

            sort(0, array.length - 1);
        }));
        
        return function () {
            array = numbers.concat();
            quickSort().start();
            return array;
        };

    })();

    for (var round = 0; round < 3; round++) {
        var iterations = 10000 * (round + 1)
        
        var now = new Date();
        for (var i = 0; i < iterations; i++) {
            sort();
        }

        console.log("Round " + round + ": " + (new Date() - now) + "ms");
    }

您无需看懂这份代码的含义，只需要关注三点即可：

* 这份代码对`eval`的使用方式即为Wind.js中唯一的使用方式，具有代表性。
* 整段被测试的代码处于一个大的闭包内部。
* `eval`内部的代码会大量访问外部的`array`代码。

这基本满足我们之前所需的要求。所有的代码可以[在GitHub上获得](https://github.com/JeffreyZhao/benchmark/tree/master/wind-jit-aot)，其中jit.js是预编译前的测试用例，而aot.js则是使用预编译器处理后的代码，可以理解为将`eval`内的代码放于原地。aot.js中不包含任何`eval`语句，因此可以完全避免`eval`对性能的影响。两个JavaScript文件都可以用Node.js直接执行，其中的test.html页面可以用浏览器打开，页面中包含JIT和AOT两个按钮，点击后便会执行相应代码并输出结果。为了避免开发面板对于脚本执行性能的影响，所有信息会直接输出在页面上。

## 实验结果

实验在我的高配的MBP with Retina Display上执行，操作系统为最新的OSX Mountain Lion，CPU信息如下：

<img src="./osx-cpu.png" width="437" />

实验结果：

<iframe width="535" height="170" frameborder="0" scrolling="no" src="https://r.office.microsoft.com/r/rlidExcelEmbed?su=-314050802145437870&Fi=SDFBA4447598B1D752!3917&ak=t%3d0%26s%3d0%26v%3d!AA1CSN1TisjyBH8&kip=1&wdAllowInteractivity=False&Item=OSX!A1%3AE7"></iframe>

图表：

<iframe width="564" height="367" frameborder="0" scrolling="no" src="https://r.office.microsoft.com/r/rlidExcelEmbed?su=-314050802145437870&Fi=SDFBA4447598B1D752!3917&ak=t%3d0%26s%3d0%26v%3d!AA1CSN1TisjyBH8&kip=1&wdAllowInteractivity=False&Item=Chart%201"></iframe>

从结果中可以看出，对于Node.js和Chrome来说，预编译前后的性能相差不多，有时甚至`eval`的代码性能会略快一些，这点在Chrome浏览器里表现尤其明显，我重复了多次实验，无论先执行哪段代码都是预编译后的代码速度稍慢，令人甚是不解。而在Safari和Firefox中，由`eval`得到的代码耗时会比预编译后的代码长约50%，可见预编译对于这两个浏览器上的确可以带来一定性能提升。

说句题外话，单比预编译后的代码，会发现Safari和Firefox浏览器的JavaScript引擎效率高于一直以“高性能”著称的V8引擎。综合之前的测试，似乎我们可以得出“浮点数运算V8更好，但普通的JavaScript代码是其他主流浏览器更快”这样的结论。

## 总结

从实验结果上看，假如您使用的是Node.js或是Chrome浏览器，那么其实预编译所获的的性能提升并不明显，而在其他浏览器中性能会有较大幅度的提升。但是在我看来，这点程度的性能提升其实也并不明显，因为Wind.js的运用场景会包含大量的异步，一个随随便便的异步操作便能抵消数以千万次甚至更多的纯计算。综合来看，`eval`完全可以用于开发环境，借助`eval`所实现的Wind.js动态编译，其带来的方便性远远胜过那么一点性能损失。

那么我们为什么还要提供预编译功能呢？因为无论如何`eval`还是会引起程序加载时的性能开销，也会对带来无谓的内存占用。而更重要的是，预编译后的代码可以摆脱体积巨大的编译器模块，让浏览器只需加载总大小为4K的脚本（Minified + GZipped）便可正常运行。