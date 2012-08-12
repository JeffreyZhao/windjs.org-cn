---
layout: page
title: Wind.js异步模块：取消模型
---

{% render_partial docs/async/_index.md %}

## 取消模型

取消操作也是异步编程中十分常见，但也十分麻烦的部分。因此，Wind.js异步模块在任务模型中融入一个简单的取消功能，丰富其潜在功能及表现能力。

但是，Wind.js的Task对象上并没有一个类似`cancel`这样的方法，这点可能会出乎某些人的意料。在实现“取消模型”这个问题上，我们首先必须清楚一点的是：**并非所有的异步操作均可撤销**。有的任务一旦发起，就只能等待其安全结束。因此，我们要做的，应该是“要求取消该任务”，至于任务会如何响应，便由其自身来决定了。在Wind.js的异步模型中，这个“通知机制”便是由`Wind.Async.CancellationToken`类型（下文也会称作CancellationToken）提供的。

CancellationToken的`cancel`方法便用于“取消”一个（或一系列）的异步操作。更准确地说，它是将自己标识为“要求取消”并“通知”相关相关的异步任务。这方面的细节将在后续章节中讲解，目前我们先来了解一下Wind.js异步模块中的任务取消模型。如果您要取消一个任务，怎么需要先准备一个CancellationToken对象：

    var ct = new Wind.Async.CancellationToken();

然后，对于支持取消的异步任务，都会接受一个CancellationToken对象作为参数，并根据其状态来行动。这里我们还是以[辅助方法](./helpers.html)中的`sleep`方法进行说明：

    var printEverySecondAsync = eval(Wind.compile("async", function (ct) {
        var i = 0;
        while (true) {
            $await(Wind.Async.sleep(1000, ct));
            console.log(i++);
        }
    }));

    printEverySecondAsync(ct).start();

如果您在浏览器或是Node.js的JavaScript交互式控制台上运行上述代码，将会从0开始，每隔一秒打印一个数字，永不停止，直到有人调用`ct.cancel()`为止。

在一个Wind.js异步方法中，“取消”的表现形式为“异常”。例如，在`ct.cancel()`调用之后，上述代码的中的`$await(Wind.Async.sleep(1000, ct))`语句将会抛出一个`Wind.Async.CanceledError`错误，我们可以使用`try…catch`进行捕获：

    var ct = new CancellationToken();
    var task = Wind.Async.sleep(5000, ct);
    try {
        setTimeout(function () { ct.cancel() }, 1000); // 1秒后调用ct.cancel();
        $await(task);
    } catch (ex) {
        console.log(CancelledError.isTypeOf(ex)); // true
        console.log(task.status); // canceled
    }

如果某个Task对象抛出了`CancelledError`错误对象，则它的`status`字段会返回字符串`"canceled"`，表明其已被取消。同理，对于一个Wind.js异步方法来说，如果从内部抛出一个未被捕获的`CancelledError`错误对象，则它的状态也会标识为“已取消”。试想，在很多情况下，我们不会用`try…catch`来捕获一个`$await`指令所抛出的异常，于是这个异常会继续顺着“调用栈”继续向调用者传递，于是相关路径上所有的Task对象都会成为`canceled`状态。这是一个**简单而统一**的模型。

有些情况下我们也需要手动操作CancellationToken对象，向外抛出一个`CanceledError`错误，以表示当前异步方法已被取消：

    if (ct.isCancellationRequested) {
        throw new Wind.Async.CanceledError();
    }

或直接：

    ct.throwIfCancellationRequested();

`throwIfCancellationRequested`是CancellationToken对象上的辅助方法，其实就是简单地检查`isCancellationRequested`字段是否为true，并抛出一个`CanceledError`对象。

有时候我们也需要手动判断`isCancellationRequested`，因为可能我们需要在取消的时候做一些“收尾工作”，于是便可以：

    if (ct.isCancellationRequested) {

        // 做些收尾工作

        throw new Wind.Async.CanceledError();
    }

或是：

    try {
        $await(…); // 当任务被取消时
    } catch (ex) {
        if (CancelledError.isTypeOf(ex)) { // 取消引发的异常
            // 做些收尾工作
        }

        throw ex; // 重新抛出异常
    }

值得注意的是，由于JavaScript的单线程特性，一般只需在异步方法刚进入的时候，或是某个`$await`指令之后才会使用`isCancellationRequested`或是`throwIfCancellationRequested`。我们没有必要在其他时刻，例如两个`$await`指令之间反复访问这些成员，因为它们的行为不会发生任何改变。

## Wind.Async.CancellationToken类型详解

实现任务取消功能时离不开`Wind.Async.CancellationToken`类型，它有以下成员：

### register(handler)

该方法用于注册一个取消时的回调方法`handler`，它会在`cancel`方法调用时执行。如果`cancel`已经调用过，则`handler`会立即执行。

### unregister(handler)

该方法用于去除取消时的回调方法`handler`，它是`register`方法的相反操作。

### cancel()

该方法用于发出取消请求，调用后会将`isCancellationRequested`字段设为true，并执行所有已注册的取消回调方法。取消后，所有的回调方法会被释放，CancellationToken对象不会保留对取消回调方法的引用。

### isCancellationRequested

该字段表示是否已经提出取消请求。

### throwIfCancellationRequested()

该方法用于在取消请求已经提出的情况下抛出一个`isCancellation`为true的错误对象。它是一个方便开发者使用的辅助方法，简单等价为：

    if (this.isCancellationRequested) {
        throw new Wind.Async.CancelledError();
    }
    
关于如何将其他异步模型融入Wind.js，使其等到得到`$await`的眷顾，请参考“[绑定其他异步模型](./binding.html)”相关内容。