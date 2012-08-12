---
layout: page
title: Wind.js异步模块：异步方法
---

{% render_partial docs/async/_index.md %}

## 定义异步方法

在JavaScript里定义一个普通方法很容易，例如：

    var print = function (text) {
        console.log(text);
    }

而创建一个Wind.js里的“异步方法”，其实就是使用“异步构造器”（即`"async"`）来创建一个Wind.js方法，即：

    var printAsync = eval(Wind.compile("async", function (text) {
        console.log(text);
    }));

其中`eval(Wind.compile("async", …))`部分只是一个包装，一种标识，让程序员可以清楚的意识到这不是一个普通方法。在使用Wind.js的时候，我们不会在其他任何地方，其他任何一种方式使用`eval`或是`Wind.compile`，完全不会表现出其“邪恶”的一面，更不会出现在生产环境中。

## 使用异步方法

### 直接使用

与普通使用不同的是，异步方法在执行后并不会立即启动方法体内的代码，而是会返回一个`Wind.Async.Task`类型的对象（后文也会称做“任务对象”或是Task对象）：

    // 输出“Hello World”
    print("Hello World");

    // 得到Wind.Async.Task对象
    var task = printAsync("Hello World");

如果要启动这个task对象，只需调用其`start()`方法即可：

    task.start(); // 输出“Hello World”

关于`Wind.Async.Task`类型成员及其功能，请参考[任务模型](./task.html)相关内容。任何一个Wind.js异步方法，都可以使用`start()`方法启动，无需依赖其他异步方法（见下节）。这点经常被人忽略，但对于那些需要将Wind.js与普通代码配合使用的场景十分重要。

### 在其他异步方法内使用

对于一个Wind异步方法返回的Task对象来说，最常见的使用方式便是在另一个异步方法内，通过`$await`命令来执行并等待其结束。例如：

    var printAllAsync = eval(Wind.compile("async", function (texts) {
        for (var i = 0; i < texts.length; i++) {
            $await(printAsync(texts[i])); // 使用$await命令执行一个Task对象
        }
    }));

当然，对于这里的`printAsync`方法来说，由于其内部并没有其他异步操作（您可以简单的理解为“没有`$await`命令”），因此它所返回的Task对象，其执行效果和普通的`print`方法没有什么区别。但是在实际使用时，我们使用的是更有意义的异步操作，例如各式[辅助方法](./helpers.html)中的`Wind.Async.sleep`方法：

    var printEverySecond = eval(Wind.compile("async", function (texts) {
        for (var i = 0; i < texts.length; i++) {
            $await(Wind.Async.sleep(1000)); // “暂停”一秒
            console.log(text[i]);
        }
    }));

我们知道，JavaScript环境并没有提供“暂停”或是“阻塞”一段时间这样的方法，唯一能提供“延迟执行”功能的只有`setTimeout`。而`setTimeout`只能做到在一段时间后执行一个回调方法，因此便无法在一个使用JavaScript中的`for`或是`while`关键字来实现循环，也无法靠`try…catch`来捕获到回调函数出错时抛出的异常。

而在Wind.js异步方法中，这一切都不是问题。`$await`将为您保证异步操作的执行顺序，您可以使用最传统的编程方式来表达算法，由Wind.js来帮您搞定异步操作的各种问题。关于`$await`的详细语义，请参考“[$await指令](./await.html)”相关内容。