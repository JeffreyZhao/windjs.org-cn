---
layout: page
title: 排序动画（异步模块示例）
---

{% render_partial docs/async/samples/browser/sorting-animations/_index.md %}

## 问题描述

排序算法是解决现实问题过程中最常见的算法之一，其中`Array`类型自带的`sort`方法，则是我们在JavaScript编程中排序一个数组时最常用的手段。

“[冒牌排序](http://zh.wikipedia.org/wiki/%E5%86%92%E6%B3%A1%E6%8E%92%E5%BA%8F)”是各种排序算法中最简单的一个，思路清晰，易于实现，因此也是编程学习过程中必然会接触的一个算法。在JavaScript中实现一个冒泡排序只需寥寥数行代码：

    var compare = function (x, y) {
        return x - y; 
    }

    var swap = function (a, i, j) {
        var t = a[i]; a[i] = a[j]; a[j] = t;
    }

    var bubbleSort = function (array) {
        for (var i = 0; i < array.length; i++) {
            for (var j = 0; j < array.length - j - 1; j++) {
                if (compare(array[j], array[j + 1]) > 0) {
                    swap(array, j, j + 1);
                }
            }
        }
    }
所谓冒泡排序，便是使用双重循环，两两比较相邻的元素，如果顺序不对，则交换两者。我们有了基本算法，想要将其用动画表现出来，其实只要运用以下两点策略即可：

* 增加交换和比较操作的耗时，这样在动画演示时能比较容易看清楚，因为排序算法的性能主要取决于交换和比较操作的次数多少。
* 每次交换元素后重绘数组，这样便能表现出排序的动画效果。

其结果便是如此：

<iframe src="http://repository.jscex.info/master/samples/async/browser/sorting-animations.html" frameborder="0" width="350" height="350"></iframe>

看上去很简单，不是吗？

## 异步编程之殇

在其他一些语言里，我们往往可以使用`sleep`函数让当前线程停止一段时间，这样便起到了“等待”的效果。但是，在JavaScript中我们无法做到这一点，唯一的“延时”操作只能使用`setTimeout`来实现，但它却需要一个回调函数，我们无法这样让`compare`方法“暂停”一段时间：

    var compare = function (x, y) {
        setTimeout(function () {
            return x - y;
        }, 10); // compare方法依然会立即返回
    }

我们只能把`compare`改造为带有回调函数的方法：

    var compare = function (x, y, callback) {
        setTimeout(function () {
            callback(x - y); // 通过回调函数传递结果
        }, 10);
    }

同理，`swap`方法也需要通过回调函数传递结果。此时我们会发现，我们很难在`bubbleSort`中使用异步的`compare`和`swap`方法，而且如果要配合循环和判断一齐使用则更加困难。这就是异步编程在流程控制方面的难点所在：我们无法使用传统的JavaScript进行表达，算法会被回调函数分解地支离破碎。

但是基于Wind.js便可以[很轻松地解决这个问题]([./solution.html])。