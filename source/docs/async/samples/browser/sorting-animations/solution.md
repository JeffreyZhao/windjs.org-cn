---
layout: page
title: 排序动画：解决方案
---

{% render_partial docs/async/samples/browser/sorting-animations/_index.md %}

## 引入Wind.js异步模块

{% render_partial docs/async/importing/_browser.md %}

## 解决方案

为了解决异步编程中的流程控制问题，人们设计构造了[各式各样的辅助类库](https://github.com/joyent/node/wiki/modules#wiki-async-flow)来简化开发难度。但在Wind.js看来，流程控制是一个语言层面上的问题，JavaScript已经提供了流程控制需要的所有关键字（例如`if`、`for`、`try`等等），开发人员也早已无数次证明了这种方式的灵活及高效。如果我们可以“修复”这些流程控制机制对异步操作“无效”的问题，则开发人员无需学习新的API，不会引入额外的噪音，一切都是最简单，最熟悉的JavaScript代码。

Wind.js便做到了这一点。例如，使用Wind.js来实现冒泡排序动画，则只需要：

    var compareAsync = eval(Wind.compile("async", function (x, y) {
        $await(Wind.Async.sleep(10)); // 暂停10毫秒
        return x - y; 
    }));

    var swapAsync = eval(Wind.compile("async", function (a, i, j) {
        $await(Wind.Async.sleep(20)); // 暂停20毫秒
        var t = a[i]; a[i] = a[j]; a[j] = t;
        paint(a); // 重绘数组
    }));

    var bubbleSortAsync = eval(Wind.compile("async", function (array) {
        for (var i = 0; i < array.length; i++) {
            for (var j = 0; j < array.length - i - 1; j++) {
                // 异步比较元素
                var r = $await(compareAsync(array[j], array[j + 1]));
                // 异步交换元素
                if (r > 0) $await(swapAsync(array, j, j + 1));
            }
        }
    }));
    
与之前的代码相比，基于Wind.js编写的代码几乎只有两个变化：

1. 与传统的`function () { ... }`方式不同，我们使用`eval(Wind.compile("async", function () { ... }))`来定义一个“异步函数”。这样的函数定义方式是“模板代码”，没有任何变化，可以认做是“异步函数”与“普通函数”的区别。
2. 对于“异步操作”，如上面代码中的`Wind.Async.sleep`内置函数（其中封装了`setTimeout`函数），则可以使用`$await(...)`来等待其完成，方法会在该异步操作结束之后才继续下去，其执行流程与普通JavaScript没有任何区别。

完整代码请参考“[排序算法动画](https://github.com/JeffreyZhao/wind/blob/master/samples/async/browser/sorting-animations.html)”示例，其中实现了“冒泡排序”，“选择排序”以及“快速排序”三种排序算法的动画。您也可以[直接体验其最终效果](http://repository.windjs.org/master/samples/async/browser/sorting-animations.html)。