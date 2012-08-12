---
layout: page
title: Wind.js异步模块：辅助方法
---

{% render_partial docs/async/_index.md %}

## 异步方法

辅助方法中的一大类为可以直接使用的异步方法。这些异步方法都定义在`Wind.Async`模块之上。

### sleep(delay, [ct])

`sleep`方法用于将当前的异步方法暂停一段时间。该方法接受两个参数：

1. `delay`：表示暂停时长，单位为毫秒。
2. `ct`：可选参数，用于取消暂停操作的CancellationToken对象。

该异步方法没有返回值。

使用示例：

    var ct = new Wind.Async.CancellationToken();

    var printEverySecondAsync = eval(Wind.compile("async", function (texts, ct) {
        for (var i = 0; i < texts.length; i++) {
            $await(Wind.Async.sleep(1000, ct));
            console.log(texts[i]);
        }
    }));

### onEvent(target, eventName, [ct])

`onEvent`方法用于监听某个对象上某个事件的“下一次”触发。该方法接受三个参数：

1. `target`：目标对象。
2. `eventName`：事件名。
3. `ct`：可选参数，用于取消监听操作的CancellationToken对象。

为了保证兼容性，`onEvent`会使用目标对象上的`addEventListener`、`addListener`或是`attachEvent`方法来监听事件，并在操作结束或是取消之后使用`removeEventListener`、`removeListener`或是`detachEvent`方法来取消监听。`onEvent`将会返回事件对象，即事件触发时传递给监听器的第一个参数。

使用示例：

    var Async = Wind.Async;
    var ct = new Async.CancellationToken();

    var drawLinesAsync = eval(Wind.compile("async", function (board, ct) {
        var currPoint = $await(Async.onEvent(board, "click", ct));

        while (true) {
            var nextPoint = $await(Async.onEvent(board, "click", ct));
            
            drawLine(
                { x: currPoint.clientX, y: currPoint.clientY },
                { x: nextPoint.clientX, y: nextPoint.clientY });
                
            currPoint = nextPoint;
        }
    }));

## 任务协作

Wind.js异步模块也包含了一些常见的任务协作辅助方法，它们作为静态方法定义在`Wind.Async.Task`类型上。

### whenAll(tasks)

`whenAll`方法接受一个Task对象的集合，该集合可以是一个数组或是键值对，甚至是数组与键值对的任意组合。`whenAll`会返回一个新的Task对象，该新对象只有在所有Task都完成（无论成功与否）之后才会结束。假如所有的输入Task对象都成功执行完毕，则新的Task对象会返回一个结果集合，其结果与输入Task集合一一对应。

使用示例：

    var Task = Wind.Async.Task;

    var getFollowersAsync = eval(Wind.compile("async", function (userId) {

        // 获取所有追随者的ID
        var followerIds = $await(getFollowerIds(userId));

        return $await(Task.whenAll({
            user: getUserAsync(userId),
            followers: followerIds.map(function (id) {
                return getUserAsync(id);
            })
        }));
        
        // 返回结果：
        // {
        //     user: <user对象>
        //     followers: [ <follower对象1>, <follower对象2>, … ]
        // }
    });
    
在启动`whenAll`返回的新Task对象时，会立即启动所有输入中尚未开始执行的Task对象。假如有一个或多个输入任务失败，则新Task对象也会处于失败状态。在这种情况下，新Task对象的`error`属性是一个`Wind.Async.AggregateError`对象，其`children`属性是一个数组，包含了所有出错Task的错误对象，但顺序不定。例如：

    var Task = Wind.Async.Task;
    var AggregateError = Wind.Async.AggregateError;
    
    var executeInParallel = eval(Wind.compile("async", function (t0, t1) {

        try {
            return $await(Task.whenAll([ t0, t1 ]));
        } catch (ex) {
            console.log(AggregateError.isTypeOf(ex)); // true
            for (var i = 0; i < ex.children.length; i++) {
                console.log(ex.children[i]);
            }
        }
    });

### whenAll(t0[, t1[, t2[, …]]])

`whenAll`方法可以接受一系列的Task对象，并返回一个新的Task对象，该新对象只有在所有Task都成功结束之后才会结束，并使用数组返回所有Task对象的结果，其顺序与输入Task的顺序一一对应。

使用示例：

    var Task = Wind.Async.Task;

    var getFollowersAsync = eval(Wind.compile("async", function (userId) {

        // 获取所有追随者的ID
        var followerIds = $await(getFollowerIds(userId));

        var results = $await(Task.whenAll(
            getUserAsync(userId),
            getFollowerListAsync(followerIds)
        ));

        return {
            user: results[0],
            followers: results[1]
        };
    });

总而言之，以下这条语句：

    Task.whenAll(t1, t2, t3, …, tN)
    
等价于：

    Task.whenAll([ t1, t2, t3, …, tN ])

### whenAny(t0[, t1[, t2[, …]]])

`whenAny`方法可以接受一系列的Task对象，并返回一个新的Task对象，该新对象会在输入的Task中任意一个完成时（无论成功失败）结束。该方法的返回值为一个对象，其`key`字段为任务在输入时的下标，其`task`对象即为第一个完成的任务。

例如，我们可以在Node.js中使用`pipe`方法传递数据流内的数据：

    var fs = require("fs");

    var streamIn = fs.createReadStream("input.txt");
    var streamOut = fs.createWriteStream("output.txt");
    streamIn.pipe(streamOut);

此时可能会有三种情况发生：

1. `streamOut`的`close`事件触发，表示正常结束。
2. `streamOut`的`error`事件触发，表示输出流异常。
3. `streamIn`的`error`事件触发，表示输入流异常。

理论上来说，三种情况每次只会发生一种，因此我们可以编写这样的代码：

    var fs = require("fs");
    var Async = Wind.Async;
    var Task = Async.Task;
    
    var copyFileAsync = eval(Wind.compile("async", function (src, target) {
        var streamIn = fs.createReadStream("input.txt");
        var streamOut = fs.createWriteStream("output.txt");
        streamIn.pipe(streamOut);
        
        var any = $await(Task.whenAny(
            Async.onEvent(streamOut, "close"),
            Async.onEvent(streamOut, "error"),
            Async.onEvent(streamIn, "error")
        ));
        
        // 如果不是第一个输入完成，则意味着出错了
        if (any.key != 0) {
            throw any.task.result;
        }
    }));

在启动`whenAny`返回的新Task对象时，会立即启动所有输入中尚未开始执行的Task对象。由于“完成”不分成功失败，因此`whenAny`返回的新对象永远不会抛出异常。

### whenAny(taskArray)

`whenAny`方法亦可接受一个Task对象数组`taskArray`作为参数，其余表现与上述重载完全相同。

使用示例：

    var fs = require("fs");
    var Async = Wind.Async;
    var Task = Async.Task;
    
    var copyFileAsync = eval(Wind.compile("async", function (src, target) {
        var streamIn = fs.createReadStream("input.txt");
        var streamOut = fs.createWriteStream("output.txt");
        streamIn.pipe(streamOut);
        
        var tasks = [
            Async.onEvent(streamOut, "close"),
            Async.onEvent(streamOut, "error"),
            Async.onEvent(streamIn, "error")];

        var any = $await(Task.whenAny(tasks));
        
        // 如果不是第一个输入完成，则意味着出错了
        if (any.key != 0) {
            throw any.task.result;
        }
    }));

### whenAny(taskMap)

`whenAny`方法的第三个重载可以接受一个对象`taskMap`作为参数，该对象上的每个字段都对应一个Task对象。与之前的两个重载相比，新Task对象的返回值中的`key`字段保存了完成的那个Task对象所对应的键值。该方法的其他表现与之前的两个重载完全相同。

使用示例：

    var fs = require("fs");
    var Async = Wind.Async;
    var Task = Async.Task;
    
    var copyFileAsync = eval(Wind.compile("async", function (src, target) {
        var streamIn = fs.createReadStream("input.txt");
        var streamOut = fs.createWriteStream("output.txt");
        streamIn.pipe(streamOut);

        var any = $await(Task.whenAny({
            end: Async.onEvent(streamOut, "close"),
            errorOut: Async.onEvent(streamOut, "error"),
            errorIn: Async.onEvent(streamIn, "error")
        }));
        
        // 如果完成的任务不是end，则意味着出错了
        if (any.key != "end") {
            throw any.task.result;
        }
    }));

## 异步操作绑定

如果要在Wind.js异步方法中使用现有的异步操作，则需要将其绑定为Wind.js中标准的Task对象。一般来说，绑定一个异步操作是一件简单直观的事情，但是像Node.js中提供了大量的异步操作，将其一一绑定也是一件不小的工作量。幸运的是，同一平台内的异步操作都基本具有相同的模式，我们可以通过编写一些的简单的辅助方法，来统一完成绑定操作。

Wind.js异步增强模块也包含了一些绑定常见异步接口的辅助方法，它们都定义在`Wind.Async.Binding`模块下面。

### fromCallback(fn)

某些异步操作会直接使用回调函数返回结果，例如Node.js中[File System模块的exists方法](http://nodejs.org/api/fs.html#fs_fs_exists_path_callback)：

    var fs = require("fs");
    fs.exists("/etc/passwd", function (exists) {
        // exists参数表示是否存在
    });

此时，便可以使用`fromCallback`将此类异步操作绑定为一个返回Task对象的异步方法：

    var Binding = Wind.Async.Binding;
    path.existsAsync = Binding.fromCallback(path.exists);

    // 某Wind.js异步方法内部
    var exists = $await(path.existsAsync("/etc/passwd"));

某些异步操作会为回调函数传入多个参数，例如：

    var split = function (numbers, n, callback) {
        var smaller = 0, larger = 0, equals = 0;
        for (var i = 0; i < numbers.length; i++) {
            var num = numbers[i];
            if (num == n) equals ++;
            else if (num > n) larger ++;
            else smaller ++;
        }
        
        callback(equals, larger, smaller);
    }

此时，也可以在`fromCallback`内逐个指定参数名称，这样最后Task对象的结果将会是一个包含这些字段的对象：

    var splitAsync = Binding.fromCallback(split, "equals", "larger", "smaller");
    
    // 某Wind.js异步方法内部
    var result = $await(splitAsync([1, 2, 3, 4, 5, 6], 3));
    console.log(result.equals); // 1
    console.log(result.smaller); // 2
    console.log(result.larger); // 3

当然，如果您只关注回调函数的第一个参数，甚至一个都不关注，自然也可以在使用`fromCallback`的时候不列出参数名称。

### fromStandard(fn)

`fromCallback`支持的是以回调形式返回结果的异步操作，而`fromStandard`返回的便是“可能会失败”的“标准接口”，它会将回调函数的第一个参数作为错误对象，如果存在这个对象，则意味着该异步操作出现了错误，于是将其当作异常对象抛出。而真正的“返回值”，则是回调函数收到的第二个参数。

例如在Node.js中，绝大部分接口都遵守了这样的模式，例如File System模块下的许多方法：

    var fs = require("fs");

    fs.readdir(path, function (err, files) { … });
    fs.stat(path, function (err, stats) { … });
    fs.mkdir("./world", function (err) { … });

此时我们便可以使用`fromStandard`创建这些异步操作的绑定：

    fs.readdirAsync = Binding.fromStandard(fs.readdir);
    fs.statAsync = Binding.fromStandard(fs.stat);
    fs.mkdirAsync = Binding.fromStandard(fs.mkdir);

绑定后的异步方法在使用时可能会抛出异常：

    try {
        $await(fs.mkdir("./hello-world"));
    } catch (ex) {
        // 出错了
    }

与`fromCallback`一样，如果回调函数拥有多个返回值（即除了第一个表示错误的参数以外，还拥有两个或以上的参数），也可以在`fromStandard`后列出参数名，例如Node.js中的Child Processes模块有个`exec`方法：

    var exec = require('child_process').exec;

    exec('ls -l', function (error, stdout, stderr) {
        if (error) {
            console.log('exec error: ' + error);
        } else {
            console.log('stdout: ' + stdout);
            console.log('stderr: ' + stderr);
        }
    });

我们可以将其绑定为：

    var execAsync = Binding.fromStandard(exec, "stdout", "stderr");

    // 某Wind.js异步方法内
    try {
        var result = $await(execAsync("ls -l"));
        console.log("stdout: " + result.stdout);
        console.log("stderr: " + result.stderr);
    } catch (ex) {
        console.log('exec error: ' + exec);
    }

当然，如果您只关注回调函数除了错误外的第一个参数，甚至一个都不关注，自然也可以在使用`fromStandard`的时候不列出参数名称。