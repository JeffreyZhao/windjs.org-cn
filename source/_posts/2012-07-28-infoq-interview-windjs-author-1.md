---
layout: post
title: "专访Wind.js作者老赵（上）：缘由、思路及发展"
date: 2012-07-28 22:01
categories: 访谈
---

## 引言

[Wind.js][windjs]是很有特点的一个JavaScript异步编程类库（其前身为Jscex），最近作者不但发布了其眼中的里程碑版（v0.6.5），还在“[我们的开源项目](http://ouropensource.51qiangzuo.com/)”系列活动和“[阿里技术嘉年华](http://adc.taobao.com/carnival/)”上连续露脸，获得广泛关注。Wind.js的作者[赵劼](http://blog.zhaojie.me/)就在本站担任编辑，InfoQ自然抓住机会对老赵做了正式的书面采访。采访篇幅较长，将分上下集播出。老赵将在上篇中借机倾述他对于库设计的思考和心得。

注：[本文首发于InfoQ](http://www.infoq.com/cn/articles/interview-jscex-author-part-1)，出于阅读体验等方面考虑，现在重新发于博客。

<!-- more -->

**Q：能否先介绍一下你自己？**

大家好，我是赵劼，在网上一般叫做老赵或者赵姐夫，经常沉迷于代码世界里，关注技术发展或是程序员的成长等话题，而不太愿意接触如今比较“流行”的产品设计，项目管理，产业分析等等，连所谓的“大规模应用架构设计”都不太关心——甚至还略有抵触，所以我可以说是个纯码农。之前呆过外企，呆过民企，创过业，也呆过国内的互联网企业，现在则是在做外资银行的内部系统，可以说是一块没有接触过的领域，痛并快乐着。

在工作之余，我也会十分热衷于在业余时间编写自己的开源​作品，其中Wind.js便是迄今为止我最为投入（没有之一）的项目。当然技术之外，我也喜欢烧烧菜，弹弹琴——别看我一副十足的码农范，但我也是从小学琴，标准琴童出身呢。

**Q：是什么促使你开发Wind.js这个异步编程类库的呢？**

应该说是需求与兴趣的共同驱使吧。

首先，随着Web平台地位的提升，霸占着浏览器的JavaScript语言也成为了世界上最流行的语言之一，甚至通过[Node.js][nodejs]进入了服务器编程领域。JavaScript的一个重要特性便是“不能阻塞”，这里的“不能”是指“不应该”而不是“无法”的意思（只要提供阻塞的API）。JavaScript是一门单线程语言，因此一旦有某个API阻塞了当前线程，就相当于阻塞了整个程序，所以“异步”在JavaScript编程中占有很重要的地位。异步编程对程序执行效果的好处这里就不多谈了，但是异步编程对于开发者来说十分麻烦，它会将程序逻辑拆分地支离破碎，语义完全丢失。因此，许多程序员都在打造一些异步编程模型已经相关的API来简化异步编程工作，例如Promise模型。而我开发Wind.js，其实也是出于这个目的，只不过使用的方式有些特别，效果更好。

其次，我本身一直是个编程语言爱好者，对这方面的关注经常不被人理解，被鄙视也不是一次两次了。Wind.js主要受到了F#的启发，F#是微软为.NET平台创建的一门函数式编程语言，以OCaml为基础，并加入了一些前沿的编程语言研究成果。其中很重要的一个特性便是“计算表达式”，它对于异步编程的支持令人眼前一亮。Wind.js的前身是Jscex，即JavaScript Computation EXpressions的缩写，它是F#中“[计算表达式][fsharp-computation-expressions]”特性在JavaScript编程语言上的体现，因此理论上说它不仅仅只有“异步编程”，但简化异步编程的确是它最重要的一部分功能，没有之一。

这么看来，我为JavaScript本身引入其他语言的异步编程特性也是顺理成章的事情吧。

**Q：正如你所说的那样，在简化JavaScript异步编程这个事情上其实已经有很多不同的方案（例如Promise等异步模型），那么你觉得这些现成的方案都有哪些不足之处，Wind.js是否有“重新发明轮子”的意味在里面呢？**

作为一个异步流程控制类库，Wind.js的唯一目的便是“改善编程体验”，这跟其他大大小小异步流程控制方案有着相同的目标——但是，改善的“程度”以及改善的“方式”便是Wind.js与它们最大的区别。

在传统的异步流程控制类库中，人们普遍使用构建各类API的方式来辅助异步逻辑编写。例如著名的[Step][step]类库为了解决异步操作顺序执行的问题，便引入了一个`Step`方法：

    Step(
        function readFile() {
            fs.readFile(filename, this);
        },
        function capitalize(err, text) {
            if (err) throw err;
            return text.toUpperCase();
        },
        function writeBack(err, newText) {
            if (err) throw err;
            fs.writeFile(filename, newText, this);
        },
        function complete(err) {
            if (err) throw err;
            console.log("complete");
        }
    );

在Step类库里，假如要将几个系统串联，则可以依次将多个函数顺序提供给`Step`方法，每个函数内部使用自身的`this`作为回调，或直接返回一个结果。那么这里就有一个问题，例如在一个面向对象开发的系统中，它们对`this`有着严格的依赖关系，而Step则强制使用`this`保存回调函数，那又该如何协调两者之间的矛盾？当然，对于一个有经验的JavaScript开发人员来说，解决这点自然不是问题，但始终需要“绕”，即便不影响功能，也会影响编程风格（模式）——由于类库的需要而不得不引入的编程风格（模式）。

串行如此，那么并行呢？Step又引入了`parallel`函数来实现这方面需求，换句话说，每增加一类需求，我们则往往需要增加一个甚至一套API。Step提供了“串行”、“并行”、以及“分组”三类执行方式，而功能更为丰富的[Async][async]类库则提供了十几种异步流程控制或是数据处理的辅助方法。

这就产生了矛盾：更强大的类库，则往往意味着更多的API，以及更多的学习时间，这样开发者才能根据自身需求选择最合适的方法。那么，多少API才能够满足广大人民群众的需求呢？这其实很难界定，根据80/20法则，80%的用户只需要20%的功能，但其实不同的用户群体需要的那20%可能各不相同，这几乎意味着一个类库为了满足各种多变的需求，则需要提供100%的功能，让不同的用户各取自身所需的20%功能。而事实上，由于人们总是能够不断地挖掘出新的需求，因此我们时不时需要设计并实现新的API。同时，为了保证现有代码的兼容性，旧的API也必须保留下来，于是类库的规模往往会不断扩大。

此外，API的粒度也是一个课题。粒度越大的API往往功能越强，可以通过少量的调用完成大量工作，但粒度大往往意味着难以复用。越细粒度的API灵活度往往越高，可以通过有限的API组合出足够的灵活性，但组合是需要付出“表现力”作为成本的。JavaScript在表现力方面有一些硬伤，即所谓“[语法噪音](syntactic-noise)”，例如定义函数所用的`function`关键字和大括号，函数返回值必须使用的`return`关键字等等，其实都限制JavaScript在某些场景下的表达能力，例如在[CoffeeScript][coffeescript]中的匿名函数`(x) -> x + 1`，使用JavaScript就必须写为`function (x) { return x + 1; }`。我们的确可以在JavaScript中设计出一套足够灵活，可以组合出各种流程的异步API，我也做过这方面的尝试，但最终会发现，这样设计出来的API始终无法清晰地表达出流程的真正意图。

总之，对于传统类库来说，在如何界定类库的“边界”，如何确定API的粒度等方面，都需要做出大量的权衡。所谓“权衡”便是通过牺牲一部分的利益来换取更大的收获，但可能在满足80%的需求的时候，剩余20%的用户只能无奈地离开了。

Wind.js的确是个“轮子”，但绝对不是“重新发明”的轮子。我不会掩饰对Wind.js的“自夸”：Wind.js在JavaScript异步编程领域绝对是一个创新，可谓前无来者。有朋友就评价说“在看到Wind.js之前，真以为这是不可能实现的”，因为Wind.js事实上是用类库的形式“修补”了JavaScript语言，也正是这个原因，才能让JavaScript异步编程体验获得质的飞跃。

**Q：你一直强调Wind.js完全就是JavaScript的编程，包括“编程体验”，为什么特别纠结这一点？你为Wind.js选择了比较特别的实现方式，也跟这个思路有关吗？这种实现方式的优势在哪里？**

我们现在想要解决的是“流程控制”问题，那么说起在JavaScript中进行流程控制，有什么能比JavaScript语言本身更为现成的解决方案呢？在我看来，**流程控制并非是一个理应使用类库来解决的问题，它应该是语言的职责**。只不过在某些时候，我们无法使用语言来表达逻辑或是控制流程，于是退而求其次地使用“类库”来解决这方面的问题。

这方面我可以举一个例子，且看这段简单的代码：

    for (var i = -5; i < 5; i++) {
        if (i < 0) {
            console.log(i + "是负数");
        } else {
            console.log(i + "是正整数");
        }
    }
    
但其实，只要我们提供了合适的`_for`及`_if`函数（附1），我们完全可以不用`for`及`if`关键字来实现相同的功能：

    _for (-5, 5, function (i) {
        _if (
            // 条件
            i < 0,
            // 满足条件的分支
            function () {
                console.log(i + "是负数");
            },
            // else分支
            function () {
                console.log(i + "是正整数");
            });
    });

其实只要是有一定函数式编程能力（主要是匿名函数）的语言，都可以在一定程度上通过这种形式的类库实现流程控制，只不过因为语言特性不同，各有美丑而已。不得不说，JavaScript虽然有着还算不错的函数式编程特性，但由于之前提到的硬伤，几乎无法优雅地完成这项任务。

事实上，上面演示的`_for`方法的灵活程度与`for`语句还有很大差距，例如，您不妨尝试下如何使用类库实现如下的`for`循环：

    var iter = getIterator();
    for (var i = 0; iter.current < i++; iter.moveNext()) {
        ...
    }
    
这会直接导致我们的`_for`函数必须接受更多的匿名函数，“享受”更多JavaScript语言的硬伤。

我想说的是，即便我们“有能力”用类库实现类似的效果，但真会有人去这么做吗？使用JavaScript语言本身提供的特性进行流程控制，对于JavaScript开发人员来说，是一件天经地义，顺理成章的事情。JavaScript语言已经提供了开发人员控制流程所需的全部关键字，而且开发人员已经无数次证明了这些关键字是多么的灵活与优雅。假如，我们这里先说“假如”——假如我们可以使用传统的JavaScript语言编写异步代码，使用JavaScript来控制流程，表达逻辑，那么我们就可以避免学习大量新的API，避免遭受JavaScript中的语法噪音，我们只需编写简单而熟悉的JavaScript代码即可。

在软件开发行业有一句很著名的话：[Every abstraction is leaky][leaky-abstractions]，换句话说，任何一种抽象都是有缺陷的。事实上，各种异步编程模型都是种抽象，它们是为了实现一些常用的异步编程模式而设计出来的一套有针对性的API。但是，在实际使用过程中我们可能遇到千变万化的问题，一旦遇到模型没有“正面应对”的场景，或是触及这种模型的限制（如Step的`this`回调），开发人员往往就只能使用一些相对较为丑陋的方式来“回避问题”。Wind.js也是一种抽象，因此也不可避免的出现leaky，但Wind.js的设计思路便是将这种抽象构建为JavaScript本身。换句话说，Wind.js的能力在很大程度上是受限于JavaScript语言的表达能力，而传统异步编程类库则是受限于自身设计的API的表达能力。哪种做法可以应对更大范围的场景，我相信应该已经不言而喻了。

所以Wind.js选择的是这么一种方式：开发人员使用最普通不过的JavaScript来表达一段逻辑，只不过通过某种方式来指定某一个环节是“异步操作”，代码执行流程会在这里“暂停”，等待该异步操作结束，然后再继续执行后续代码。如果这个操作失败，则相当于抛出了一个异常，会被`catch`语句捕获。一段类似的代码可以是这样的：

    try {
        var content = $await(read(src, content));
        console.log("内容读取成功");
        $await(write(target, content));
        console.log("内容写入成功");
    } catch (ex) {
        $await(submitError(ex));
        console.log("错误提交成功");
    }

在这段代码里，我们用`$await(<task>)`标记一个异步操作，代码流程会在此处“等待”异步操作完成，在`catch`分支中也一样。我想，任何一个普通的JavaScript程序员都能顺利理解这段代码的含义。至于性能，自然必须和使用传统回调风格写出来的程序完全一致，这里的“等待”并不是“阻塞”，而会空出执行线程，直至操作完成。而且，假如系统本身没有提供阻塞的API，我们甚至没有“阻塞”代码的方法（当然，本就不该阻塞）。

当然，单纯依靠JavaScript语言本身显然无法获得这样的能力，JavaScript的执行流程不会随便地暂停，因此我们需要对代码进行**自动改写**。这便是Wind.js的工作原理，值得注意的是“自动”二字，换句话说开发者编写代码的时候完全不需要额外的“改写”步骤，这个功能在Wind.js中叫做“[JIT（Just in Time）编译][jit]”，也就是在程序的执行过程中动态生成并执行新的代码。

Wind.js自动改写的最小单位是“函数”，传统JavaScript函数是这样定义的：

    var func = function () {
        // 函数体
    };
    
而Wind.js函数是这样定义的：

    var windFunc = eval(Wind.compile("async",
        function () {
            // Wind.js函数体
        }
    ));

可见，在Wind.js在使用形式上和一个普通类库毫无二致，这便保证了它与JavaScript有着相同的编程体验。传统如CoffeeScript或是[Dart][dart]等以JavaScript作为目标语言的“新语言”，都必须引入一个额外的编译步骤。我反复强调，使用Wind.js进行开发与普通的JavaScript编程几乎没有任何区别。至于外围的“架子代码”，我们完全可以不关心其特别含义，作为代码片段加入编辑工具，敲一个快捷键便能直接输出。

在Wind.js函数中，我们就可以使用唯一的语言扩展：“绑定”操作，这可以简单理解为那个异步的`$await`操作。可以看到，这个特殊操作使用了JavaScript的“函数调用”语义，同样符合JavaScript的编程实践。有些朋友建议把`$await`改成关键字形式，而不是函数调用。可惜，这么做首先会被JavaScript运行环境拒之门外，导致无法实现JIT编译器，其次也有可能会破坏一些相关的事物，例如开发工具里的代码着色，甚至某些高级IDE中的语法提示，因为语法提示功能需要先对代码语义进行完整的分析。因此，保证Wind.js与JavaScript绝对一致，其实也是在保证Wind.js能够完美融入整个JavaScript生态环境。

不仅是语法设计上如此，Wind.js自带的类库设计也充分借助了JavaScript的语义实现。例如Wind.js的异步任务模型支持取消操作，要将一个任务置为“取消”状态，只需抛出一个未经捕获的`CanceledError`错误即可，例如：

    var t1 = eval(Wind.compile("async", function () {
        $await(t2());
    }));
    
    var t2 = eval(Wind.compile("async", function () {
        $await(t3());
    }));
    
    var t3 = eval(Wind.compile("async", function () {
        throw new CanceledError();
    }));

根据JavaScript的语义，一个未捕获的异常会顺着调用链不断抛出，导致一系列的异步任务都会进入“取消”状态。这是一个统一而简单的取消模型。

**Q：Wind.js原来的版本API极其简单，可以说只有一个$await。发展到现在，Wind.js增加了一些扩展，比如针对Node.js、WinRT的扩展，还有编程模型方面的扩展，像seq、agent这些。这些扩展是否说明Wind.js的应用场景超出你原来的预想？Wind.js的surface API和语法有没有因为这些扩展而变得复杂呢？**

说实话，Wind.js目前还处于“叫好不叫座”的推广期，并没有收到太多额外的用户需求，所有的组件与都是由我设计并实现的，所以当然都在我预想之内啦！

其实你说的这些概念都不是同一个层面上的东西，它们可以分两大类，一是用于不同目的的“构造器”：

* [异步][windjs-async]：即`async`构造器，用于简化异步开发，基于Wind.js自带的异步任务模型。
* 迭代：即`seq`构造器，用于生成一个延迟计算的迭代器。
* [Promise][windjs-promise]：即`promise`构造器，用于支持基于Promise/A模型的异步开发。

“构造器”和“编译器”是Wind.js的两大支柱。Wind.js编译器在转化一个普通的JavaScript函数的时候永远使用相同的模式，但配合不同的“构造器”便能得到不同的效果。例如`seq`构造器可以用来创建一个无限长的迭代器：

    var infinite = eval(Wind.compile("seq", function (start) {
        while (true) {
            $yield(start++);
        }
    }));

    var fib = eval(Wind.compile("seq", function () {
        $yield(0);
        $yield(1);

        var a = 0, current = 1;
        while (true) {
            var b = a;
            a = current;
            current = a + b;

            $yield(current);
        }
    }));

不同的构造器中的“绑定”操作不同，其含义也不一样。例如在`seq`构造器中的绑定操作为`$yield`，表示输出迭代器里的一个对象，并将控制权交由外部，直到下次调用`moveNext`方法时才继续执行。因此，虽然上述两个方法都是会输出“无限长”的序列，但什么时候取元素，取多少元素完全由使用者控制，即所谓“延迟计算”，“按需计算”。

两个迭代器还能够组合使用，例如：

    fib() // 无限长的菲波纳契数列
        .filter(function (n) { return n % 2 == 0; }) // 取其中所有的偶数
        .skip(2) // 跳开前2个数
        .take(5) // 取5个数
        .zip(infinite(1)) // 和从1开始的无限长序列组合
        .map(function (a) { return a[1] + ": " + a[0]; }) // 将每个组合拼接成字符串
        .each(print); // 并打印出来

即便是无穷序列，但由于我们使用了`take(5)`操作，最终也只会打印出五项，而不会成为死循环。这类“组合”操作便是“增强类库”的功劳。面向异步编程的`async`构造器也有相关的增强类库，它扩展了`async`构造器所使用的异步任务模型，例如并发相关的辅助函数。此外还有一些常用的异步操作（如封装了`setTimeout`的`sleep`方法），并简化了与其他一些常见异步编程模型的协作。你所提到的Node.js以及agent扩展都是类似的“类库”，前者自不必说，后者则是一个Erlang中Actor模型的尝试实现，基于Wind.js的异步编程支持。

不过你提到的WinRT扩展就不同了，它其实便是`promise`构造器，是为了直接支持使用[Promise/A][promise]异步模型的开发环境。Promise/A现在是CommonJS的草案之一，提出了一种Promise模型的设计及API表现。虽说它离“标准”还有很长一段距离，但其实很多类库都已经实现了这个规范了，例如著名的[jQuery][jquery-promise]，[node-promise][node-promise]，还有用来编写Win8中Metro应用（即WinRT）的HTML5开发平台。当然严格来说，它们都是基于Promise/A规范的一套“扩展实现”，但既然有了共有的子集，那事情就已经好办多了。

之前在一个QQ群上某同学建议我提供一个类似于`fromStandard`（用于将标准的回调异步方法适配为Wind.js异步模型）一样的`fromPromise`辅助方法。这当然没问题，其实很简单，接下来也会做，但Wind.js考虑得更多，例如：

1. 为什么需要`fromPromise`辅助方法？因为用户使用了Promise异步模型，而希望Wind.js提供更好的辅助环境。
2. 为什么对方不使用Wind.js自带的异步任务模型？因为用户可能已经有部分代码采用了Promise模型。
3. 为什么它要使用Promise这种已经较为成熟且复杂的异步模型？因为用户可能已经有了一个围绕着Promise模型开发的应用程序，甚至是一个已经拥有大量辅助方法支持的应用开发框架（例如WinRT）。

因此，这种情况下如何还需要再结合Wind.js的异步任务模型，则需要来回转换，显得略为冗余——不如就让Wind.js对Promise/A异步模型直接提供支持吧！换句话说，要让`$await`操作接受一个Promise对象而不是一个Wind.js异步任务对象，同时一个使用`promise`构造器生成的Wind.js异步方法也会直接返回一个Promise/A对象。于是我们在WinRT中便可以编写这样的代码：

    WinJS.Namespace.define("MyApp", {
        showPhoto: eval(Wind.compile("promise", function () {
            var dlg = new MessageDialog("Do you want to open a file?");
            dlg.commands.push(new UICommand("Yes", null, "Yes"));
            dlg.commands.push(new UICommand("No", null, "No"));

            // 显示对话框，并等待用户选择Yes或No
            var result = $await(dlg.showAsync());

            if (result.id == "Yes") {
                var picker = new FileOpenPicker();
                picker.viewMode = PickerViewMode.thumbnail;
                picker.suggestedStartLocation = PickerLocationId.picturesLibrary;
                picker.fileTypeFilter.push(".jpg");

                // 显示文件选择器，并等待用户选择文件
                var file = $await(picker.pickSingleFileAsync());

                if (file != null) {
                    $("#myImg").src = URL.createObjectURL(file);
                }
            }
        }))
    });

此时我们可以一视同仁地对待showPhoto方法以及WinRT中标准的异步操作，甚至可以把它传入WinRT的系统方法配合使用，因为它完全符合WinRT标准。值得一提地是，创建`promise`构造器只需花费30余行代码（包括空行及函数定义）。从这个例子中可以看出，在需要的情况下，我们完全可以轻松地让Wind.js融入任何异步模型。

一开始我就提到，Wind.js其实是受到F#计算表达式的启发而诞生的JavaScript类库，这些所谓的“变化”和“扩展”实际上都没有逃离计算表达式及其基础理论[Monad][monad]的范畴。Wind.js从诞生开始便一直差不多是现在这样的使用方式，只是其内部实现在不断完善而已。不过我目前的精力完全放在了异步编程上，因此如`seq`构造器或是agent扩展都还处于孵化阶段，虽然它们几乎是我还在验证Wind.js可行性的阶段时开发的。

## 附1

我们可以这样实现`_for`及`_if`函数：

	var _for = function (begin, end, body) {
	    for (var i = begin; i < end; i++) {
			body(i);
		}
	};
	
	var _if = function (condition, bodyPart, elsePart) {
		if (condition) {
		    bodyPart();
		} else {
			elsePart();
		}
	};

## 参考

Wind.js相关：

* [网站][windjs]
* [源码][windjs-github]
* [文档][windjs-docs]
    * [编译器模块][windjs-compiler]
    * [异步模块][windjs-async]
    * [Promise模块][windjs-promise]

项目：

* [Step][step]
* [Async][async]
* [CoffeeScript][coffeescript]
* [Dart][dart]

Promise模型：

* [Promise/A规范][promise]
* [node-promise][node-promise]
* [jQuery中的Promise][jquery-promise]
* [Win8 Metro开发中的Promise][winrt-promise]

其他：

* [F#计算表达式](fsharp-computation-expressions)
* [Monad][monad]
* [语法噪音][syntactic-noise]
* [JIT编译][jit]
* [抽象泄露定律][leaky-abstractions]

[windjs]: {{root_url}}/  "Wind.js官方网站"
[windjs-github]: https://github.com/JeffreyZhao/wind  "Wind.js源码"
[windjs-docs]: {{root_url}}/docs/ "Wind.js文档"
[windjs-compiler]: {{root_url}}/docs/compiler/ "Wind.js编译器模块"
[windjs-async]: {{root_url}}/docs/async/  "Wind.js 异步模块"
[windjs-promise]: {{root_url}}/docs/promise/ "Wind.js Promise模块"
[jit]: http://en.wikipedia.org/wiki/Just-in-time_compilation "JIT编译"
[promise]: http://wiki.commonjs.org/wiki/Promises/A "Promise/A模型"
[node-promise]: http://github.com/kriszyp/node-promise "node-promise类库"
[jquery-promise]: http://api.jquery.com/promise/ "jQuery中的Promise模型"
[winrt-promise]: http://msdn.microsoft.com/en-us/library/windows/apps/br211867.aspx "WinRT中的Promise模型"
[coffeescript]: http://coffeescript.org/ "CoffeeScript语言"
[dart]: http://www.dartlang.org/ "Dart语言"
[step]: https://github.com/creationix/step/ "Step类库"
[async]: https://github.com/caolan/async/ "Async类库"
[nodejs]: http://nodejs.org/ "Node.js"
[syntactic-noise]: http://martinfowler.com/bliki/SyntacticNoise.html "语法噪音"
[fsharp-computation-expressions]: http://msdn.microsoft.com/en-us/library/dd233182.aspx "F#计算表达式"
[monad]: http://en.wikipedia.org/wiki/Monad_(functional_programming) "Monad (Wikipedia)"
[leaky-abstractions]: http://www.joelonsoftware.com/articles/LeakyAbstractions.html "Leaky Abstractions"
