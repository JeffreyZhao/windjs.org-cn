---
layout: page
title: Wind.js异步模块：任务模型
---

{% render_partial docs/async/_index.md %}

## 任务模型

`$await`指令的参数是`Wind.Async.Task`类型的对象，这个对象可以是一个异步方法的返回结果，或是由其他任何方式得到。在Wind.js异步模块眼中，一个异步任务便是指“**能在未来某个时刻返回的操作**”，它可以是一个`setTimeout`的绑定（如之前演示过的`sleep`方法），甚至是一个用户事件：

    var btnNext = document.getElementById("btnNext");
    var ev = $await(Wind.Async.onEvent(btnNext, "click"));
    console.write(ev.clientX, ev.clientY);

因为在Wind.js看来，用户的点击行为，也是“能在未来某个时刻返回的操作”，这便是一个异步任务。您可以在“[模态对话框](samples/browser/modal-dialog/)”以及“[汉诺塔](samples/browser/hanoi/)”示例中了解这个模型的威力。

除了上文提到的`sleep`及`onEvent`以外，Wind.js异步模块还提供了更多有用的[辅助方法](./helpers.html)来应对常见的异步协作问题。例如之前的“并发”执行示例，在实际情况下往往会使用`whenAll`辅助方法：

    var Task = Wind.Async.Task;

    var getUserItemsAsync = eval(Wind.compile("async", function (userId) {
        return $await(Task.whenAll({
            user: queryUserAsync(userId),
            items: queryItemsAsync(userId)
        }));
    });

`whenAll`辅助方法会将输入的多个任务包装为一个整体，并同样以Task对象的形式返回。新的Task对象只有在所有输入任务都结束的情况下才会完成，并使用相同的结构（“键值”或“数组”）返回其结果。

Wind.js的异步模型经过C#，F#等多种语言平台的检验，可以说拥有非常灵活而丰富的使用模式。

## Wind.Async.Task类型详解

`Wind.Async.Task`是Wind.js异步模块内的标准异步模型。异步方法产生的Task对象，除了可以交给`$await`指令使用之外，也可以直接使用这个对象。这种做法在某些场合十分重要，例如要在系统中逐步引入Wind.js时的情况。

### 静态 create(delegate)

该方法是Task类型上的静态方法，用于创建一个Task对象，多在将普通异步操作绑定为Task的时候使用。

参数`delegate`方法会在Task启动时（即`start`方法被调用时）执行，签名为`function (t)`，其中`t`即为此次`create`调用所返回的Task对象。

使用示例：

    Task.create(function (t) {
        console.log(t.status); // running
    });

### start()

该方法用于启动任务，只可调用一次。

使用示例：

    var task = someAsyncMethod();
    task.start();

### addEventListener(ev, listener)

该方法用于添加一个事件处理器，只能在Task对象状态为`ready`或`running`的时候添加。

参数`ev`为是以字符串表示的事件名，可以为：

* **success**：任务执行成功时触发，此时该任务的`status`字段为`succeeded`，且`result`字段为执行结果。
* **failure**：任务执行失败时触发，此时该任务的`status`字段为`faulted`或`canceled`（视错误对象的`isCancallation`字段而定），且`error`字段为错误对象。
* **complete**：无论任务成功还是失败，都会触发该事件。

参数`listener`为事件处理方法，无参数，执行时`this`为触发事件的Task对象。

使用示例：

    var task = someAsyncMethod();

    task.addEventListener("success", function () {
        console.log("Task " + this.id + " is succeeded with result: " + this.result);
    });

    task.addEventListener("failure", function (t) {
        console.log("Task " + this.id + " is failed with error: " + this.error);
    });

    task.addEventListener("complete", function (t) {
        console.log("Task " + this.id + " is completed with status: " + this.status);
    });

### removeEventListener(ev, listener)

该方法用于去除一个事件处理器，提供与`addEventListener`功能相反的操作。值得注意的是，在`complete`方法调用之后，Task对象会自动释放对事件处理器，不会继续保持对它们的引用。

### complete(type, value)

该方法用于通知该Task对象已“完成”（无论结果如何），多在将普通异步操作绑定为Task的时候使用。根据不同情况，参数的值应分别为：

* **成功**：参数`type`为`"success"`，`value`为任务的执行结果。
* **出错**：参数`type`为`"failure"`，`value`为错误对象，且该错误对象不是`Wind.Async.CanceledError`对象。
* **取消**：参数`type`为`"failure"`，`value`为错误对象，且该错误对象为`Wind.Async.CanceledError`对象。

使用示例：

    fs.readFileAsync = function (path) {
        return Task.create(function (t) {
            fs.readFile(path, function (err, data) {
                if (err) {
                    t.complete("failure", err); // 出错
                } else {
                    t.complete("success", data); // 成功
                }
            });
        });
    }

### result

该字段保存了Task对象执行**成功**后得到的结果，在异步方法内将作为`$await`指令的返回值。

### error

该字段保存了Task对象执行失败（**出错**或**取消**）后的错误对象，在异步方法内将作为异常抛出。

### status

该字段标识Task对象的状态，可分为以下几种情况：

* **ready**：创建完成，等待启动。
* **running**：正在执行。
* **succeeded**：执行成功。
* **faulted**：执行出错。
* **canceled**：执行已取消。

可见，Task对象的状态之一为“已取消”，关于这方面的更多信息请参考下一节“[取消模型](./cancellation.html)”。