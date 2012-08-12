---
layout: page
title: Wind.js异步模块：绑定其他异步模型
---

{% render_partial docs/async/_index.md %}

## 将其他异步操作绑定为Task对象

世界上有无数种异步模型，从最简单的回调函数传递结果，用户行为引发的事件，到相对复杂的Deferred模型。而能够被Wind.js异步模块中`$await`指令所识别的，便只有用`Wind.Async.Task`类型来表达的异步任务。任何的异步方法，在执行后都能得到一个Task对象，但如果是其他平台或是环境所提供异步模型，便需要经过“绑定”才能被`$await`使用。

### 绑定简单操作

将任何一个异步操作Task对象，会需要用到`Wind.Async.Task`类型的`create`静态方法。方便起见，通常我们可以使用`Task`来指向这个全命名：

    var Task = Wind.Async.Task;

例如在Node.js中[File System模块的exists方法](http://nodejs.org/api/fs.html#fs_fs_exists_path_callback)，便是一个十分简单的异步操作，它会将结果通过回调函数返回：

    fs.exists('/etc/passwd', function (exists) {
        util.debug(exists ? "it's there" : "no passwd!");
    });

但如果我们要在Wind.js异步方法里使用这个函数，则需要将其进行简单绑定：

    fs.existsAsync = function (p) {
        return Task.create(function (t) {
            fs.exists(p, function (exists) {
                t.complete("success", exists);
            });
        });
    }

于是`existsAsync`就会返回一个Task对象，它在`start`以后，便会调用原来的`exists`方法获得结果。我们也可以将其用在某个Wind.js异步方法中：

    // 某Wind.js异步方法内
    var exists = $await(fs.existsAsync("/etc/passwd"));
    util.debug(exists ? "it's there" : "no passwd!");

绑定一个异步方法的基本方式可以分为以下几点：

1. 编写一个新方法，其中返回`Task.create`的执行结果（一个Task对象）。
2. `Task.create`方法的参数为一个回调函数（下文称为委托方法），它会在这个Task对象的`start`方法调用时执行，发起被绑定的异步操作。
3. 委托方法的参数是当前的Task对象（也是之前`Task.create`创建的对象），在异步操作完成后，使用其`complete`方法通知Task对象“已完成”。
4. `complete`方法的第一个参数为字符串`"success"`，表示该异步操作执行成功，并可以通过第二个参数传回该异步操作的结果（亦可空缺）。

### 引发异常

并非所有的异步操作都会成功，在平时“非异步”的编程方式中，我们往往会在出错的情况下抛出异常。如果一个异步操作引发了异常，我们只需要在调用Task对象的`complete`方法时，将第一个参数从`"success"`替换为`"failure"`，并将第二个参数设为错误对象即可。例如Node.js中[File System模块的readFile方法](http://nodejs.org/api/fs.html#fs.readFile)便可能会失败：

    fs.readFile('/etc/passwd', function (err, data) {
        if (err) {
            util.debug("Error occurred: " + err);
        } else {
            util.debug("File length: " + data.length);
        }
    });

而将其绑定为Task对象时只需：

    fs.readFileAsync = function (path) {
        return Task.create(function (t) {
            fs.readFile(path, function (err, data) {
                if (err) {
                    t.complete("failure", err);
                } else {
                    t.complete("success", data);
                }
            });
        });
    }

于是在一个Wind.js异步方法中使用时：

    // 某Wind.js异步方法内
    try {
        var data = $await(fs.readFileAsync(path));
        util.debug("File length: " + data.length);
    } catch (ex) {
        util.debug("Error occurred: " + ex);
    }

错误处理也是异步编程的主要麻烦之处之一。在异步环境中，我们往往需要在“每个”异步操作的回调函数里判断是否出现错误，一旦有所遗漏，在出现问题之后就很难排查了。例如：

    fs.readFile(file0, function (err0, data0) {
        if (err0) {
            // 错误处理
        } else {
            fs.readFile(file1, function (err1, data1) {
                if (err1) {
                    // 错误处理
                } else {
                    fs.readFile(file2, function (err2, data2) {
                        if (err2) {
                            // 错误处理
                        } else {
                            // 使用data0，data1和data 2
                        }
                    });
                }
            });
        }
    });

如今Wind.js改变了这个窘境，只需要一个`try…catch`便可以捕获到任意多个异步操作的异常，保留了传统编程过程中的实践：

    try {
        var data0 = $await(fs.readFileAsync(file0));
        var data1 = $await(fs.readFileAsync(file1));
        var data2 = $await(fs.readFileAsync(file2));
        // 使用data0，data1和data2
    } catch (ex) {
        // 错误处理
    }

甚至，在编写绝大多数Wind.js异步方法的时候，我们并不需要显式地进行`try…catch`，我们可以让异常直接向方法外抛出，由统一的地方进行处理。这其实便是JavaScript编程的传统实践。

### 取消操作

从上文的“取消模型”中我们得知，所谓“取消”只不过是引发一个`Wind.Async.CancellationError`异常而已。因此，如果要表示当前异常操作被取消，也只需要向`complete`方法传入`"failure"`即可。不过问题的关键是，我们如果要绑定一个现有的异步操作，往往还需要在取消时实现一些“清理”工作。这里，我们便以[辅助方法](./helpers.html)中的`sleep`方法来演示“取消”操作的实现方式。

`sleep`方法绑定了JavaScript运行环境中的`setTimeout`及`clearTimeout`函数，它们的基本使用方式为：

* `var seed = setTimeout(fn, delay);`：表示在`delay`毫秒以后执行`fn`方法，并返回`seed`作为这次操作的标识，供`clearTimeout`使用。
* `clearTimeout(seed);`：在`fn`被执行之前，可以使用`clearTimeout`取消这次操作。这样即便到了时间，也不会执行fn方法了。

基于这两个功能，我们便可以实现`sleep`方法及其取消功能了。实现支持取消的异步操作绑定往往分三步进行：

    var Task = Wind.Async.Task;
    var CanceledError = Wind.Async.CanceledError;

    var sleep = function (delay, /* CancellationToken */ ct) {
		return Task.create(function (t) {
            // 第一步
			if (ct && ct.isCancellationRequested) {
                t.complete("failure", new CanceledError());
            }

            // 第二步
            var seed;
            var cancelHandler;
            
            if (ct) {
                cancelHandler = function () {
                    clearTimeout(seed);
                    t.complete("failure", new CanceledError());
                }
            }
            
            // 第三步
            var seed = setTimeout(function () {
                if (ct) {
                    ct.unregister(cancelHandler);
                }
                
                t.complete("success");
            }, delay);
            
            if (ct) {
                ct.register(cancelHandler);
            }
		});
    }

**第一步：判断CancellationToken状态。**取消操作由CancellationToken类型对象来提供，但由于其往往是可选操作，因此`ct`参数可能为`undefined`。在sleep方法一开始，我们先判断`ct.isCancellationRequested`是否为true，“是”便直接将Task对象传递“取消”信息。这是因为在某些特殊情况下，该CancellationToken已经被标识为取消了，作为支持取消操作的异步绑定，这可以算作是一个习惯或是规范。

**第二步：准备取消方法。**在这里我们准备两个变量`seed`和`cancelHandler`，前者将在稍后发起`setTimeout`时赋值。我们只在用户传入`ct`时才创建`cancelHandler`方法，该方法执行时会使用`clearTimeout(seed)`来取消已经发起的`setTimeout`操作，并通过`complete`方法将该Task对象传递“取消”信息。

**第三步：发起异步操作并注册取消方法。**接着便要发起我们绑定的异步函数了。我们将`setTimeout`后得到的标识符保留在seed变量里，供之前的`cancelHandler`使用。在`delay`毫秒后会执行的方法中，我们将注册在`ct`上的取消方法去除，并通过`complete`方法将该Task对象标识为`"success"`。发起异步操作之后，再讲取消方法注册到`ct`上。当有人调用`ct`的`cancel`方法时，该取消方法便会被执行。

将一个支持取消的异步操作绑定为Task对象是最为麻烦的工作，幸好这样的操作并不多见，并且也有十分规则的模式可以遵循。

### 辅助方法

似乎将已有的异步操作绑定为Task对象是十分耗时的工作，但事实上它的工作量并不一定由我们想象中那么大。这是因为在相同的环境，类库或是框架里，它们各种异步操作都具有相同的模式。例如在Node.js中，几乎所有的异步操作都遵循`fs.exists`和`fs.readFile`这两种模式。因此，在实际开发过程中，我们不会为大量异步操作各自实现一份绑定，而会使用统一的辅助方法来应对同一类模型。例如Wind.js模型便内置了`fromCallback`及`fromStandard`辅助方法：

    var Wind = require("wind"),
        Binding = Wind.Async.Binding,
        fs = require("fs");

    fs.existsAsync = Binding.fromCallback(fs.exists);

    fs.readFileAsync = Binding.fromStandard(fs.readFile);
    fs.writeFileAsync = Binding.fromStandard(fs.writeFile);
    
    fs.readAsync = Binding.fromStandard(fs.read, "bytesRead", "buffer");
    fs.writeAsync = Binding.fromStandard(fs.write, "written", "buffer");

这便是JavaScript语言的威力。更多相关内容请参考下一节“[辅助方法](./helpers.html)”。