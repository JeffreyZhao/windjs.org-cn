---
layout: page
title: Wind.js异步模块：$await指令
---

{% render_partial docs/async/_index.md %}

## $await指令

Wind.js函数是标准的JavaScript函数，支持JavaScript语言几乎所有特性：条件判断就用`if…else`或`switch`，错误捕获就用`try…catch…finally`，循环就用`while`、`for`、`do`，其他包括`break`，`continue`，`return`等等，都是最普通的JavaScript写法，而在Wind.js异步方法中，唯一新增的便是`$await`指令。

`$await`指令的使用形式便是普通的方法调用，但事实上在上下文中并没有这个方法。它的作用与`eval(Wind.compile("async", …))`一样，仅仅是个占位符，让Wind.js知道在这个地方需要进行“特殊处理”。

`$await`指令的参数只有一个，便是一个`Wind.Async.Task`类型的对象，这个对象可以是一个异步方法的返回结果，或是以其他任何方式得到。例如[辅助方法](./helpers.html)中的`Wind.Async.sleep(1000)`，便是返回一个表示“等待1秒钟”的Task对象。因此这句代码：

    $await(Jscex.Async.sleep(1000));

在需要时也可以分开写作：

    var task = Jscex.Async.sleep(1000);
    $await(task)

`$await`指令的确切语义是：“**等待该Task对象结束（返回结果或抛出错误）；如果它尚未启动，则启动该任务；如果已经完成，则立即返回结果（或抛出错误）**”。因此，我们也可以在需要的时候灵活使用`$await`指令。例如在一个Node.js应用程序中，时常会实现下面的逻辑：
    
    var getUserItemsAsync = eval(Jscex.compile("async", function (userId) {

        var user = $await(queryUserAsync(userId));
        var items = $await(queryItemsAsync(userId));

        return {
            user: user,
            items: items
        };
    });

在上面的代码中，`queryUserAsync`与`queryItemsAsync`是依次执行的，如果前者耗时200毫秒，后者耗时300毫秒，则总共需要500毫秒才能完成。但是，在某些情况下，我们可以让两个操作“并发”执行，例如：

    var getUserItemsAsync = eval(Jscex.compile("async", function (userId) {

        var queryUserTask = queryUserAsync(userId);
    
        // 手动启动queryUserAsync任务，start方法调用将立即返回。
        queryUserTask.start();

        var items = $await(queryItemsAsync(userId));
        var user = $await(queryUserTask); // 等待之前的任务完成

        return {
            user: user,
            items: items
        };
    });

在`$await(queryUserTask)`时，如果该任务已经完成，则会立即返回结果，否则便会等待其完成。因此，当这两个互不依赖的查询操作并发执行的情况下，总耗时将会减少到300毫秒。不过，在实际情况下假如要并行两个异步操作，往往会使用`whenAll`辅助方法。

关于`Wind.Async.Task`类型的详细信息，可参考下一节“[任务模型](./task.html)”相关内容。
