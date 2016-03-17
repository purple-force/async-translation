## 你不知道JS：异步

## 第一章：异步：现在 & 以后（Now & Later）

在像JavaScript这种编程语言中，最重要也是常被误解的一点是如何表示和控制程序的行为，使它扩展到一段时间内。

这并不只是从`for`循环开始到`for`循环结束所发生的那么简单，当然，它会花费一定的时间（几微秒到几毫秒）。我们关注的是当你的部分程序现在运行，而另一部分之后运行时所发生的事情--在 *现在* 和 *以后* （对应的程序并不是主动执行的）之间有个界限。

日常开发中，所有写过的优秀程序（尤其是在JS中）都或多或少地需要管理这个界限，不论是等待用户输入、从数据库或文件系统请求数据、跨网络发送数据和等待响应，还是定期执行一个重复任务（比如动画）。在所有这些场景当中，你都得及时地在界限间管理状态。正如伦敦地铁著名的那句：小心间隙（mind the gap）。

事实上，程序中的 *现在* 和 *以后* 的关系是异步编程的核心。

可以确认的一点是，自出现JS，异步编程就一直贯穿始终。但绝大多数JS开发者没有认真考虑过，他们的程序中是怎么以及为什么突然出现异步的，或者研究过如何用其它方法实现它。回调函数一直都是足够好的方法。至今仍有许多人坚持认为回调函数已经足够了。

但是，为了满足作为运行在浏览器端和服务端以及每个能想象的到的设备中的第一等语言的广泛的需求，JS在范围和复杂度上面都在不断增长，我们管理异步的痛苦与日俱增。所以需要寻求一种功能更好、更合理的方法。

尽管目前看起来十分抽象，随着不断深入这本书，我向你们保证我们能够更完整和彻底地处理异步。在之后几章中，我们会探索JS中出现的各种异步编程技术。

但在开始之前，我们必须深刻理解下什么是异步，以及如何在JS中实现。

### 块状形式的程序（A Program in Chunks）

你可能在一个`.js`的文件中写JS程序，但是你的程序绝大多数是由几块组成的，只有一个是 *现在* 执行的，其余的是 *以后* 执行的。最常见的块单元是`function`。

绝大多数JS新手遇到的问题是认为 *以后* 不是严格地在 *现在* 之后立即发生的。换句话说，从定义上讲，不能 *现在* 完成的任务将会异步完成，因此不会出现你直观上期望或想要的阻塞行为。

考虑如下代码：


```javascript
// ajax(..) is some arbitrary Ajax function given by a library
var data = ajax( "http://some.url.1" );

console.log( data );
// Oops! `data` generally won't have the Ajax results
```
你也许注意到了标准的Ajax请求并不是同步的，这意味着`ajax(...)`还没有返回值用来赋给`data`变量。如果`ajax(...)`能够阻塞程序直到返回响应，则`data=...`赋值操作能够正常运行。

但这不是我们运用Ajax的方式。我们 *现在* 作了一个异步的Ajax请求，直到 *以后* 才获得结果。

最简单的（但显然不是唯一，更不是最好的）的从 *现在* 等直到 *以后* 的方法是采用一个函数，通常称为回调函数：

```javascript
// ajax(..) is some arbitrary Ajax function given by a library
ajax( "http://some.url.1", function myCallbackFunction(data){

    console.log( data ); // Yay, I gots me some `data`!

} );
```
**警告：** 你可能听说过可以使用同步Ajax请求。尽管技术上是可行的，但是在任何情况下绝对不要使用。因为它会锁死浏览器UI（按钮、菜单、滚动等），并且会阻止任何用户交互。这是个很糟糕的想法，应极力避免。

在你抗议争论之前，你所期望的避免回调混乱不是为同步Ajax造成的阻塞辩护。

例如，考虑以下代码：

```javascript
function now() {
    return 21;
}

function later() {
    answer = answer * 2;
    console.log( "Meaning of life:", answer );
}

var answer = now();

setTimeout( later, 1000 ); // Meaning of life: 42
```
这段程序有两大块(chunk)，一块是 *现在* 执行的，另一块是 *以后* 执行的。这两块应该很清楚就能看出来，让我们明确解释一下：

*现在*：

```javascript
function now() {
    return 21;
}

function later() { .. }

var answer = now();

setTimeout( later, 1000 );
```

*以后*：

```javascript
answer = answer * 2;
console.log( "Meaning of life:", answer );
```
只要你执行程序，*现在*块立即运行。但是`setTimeout(...)`也设置了一个事件（一个超时），用来*以后*执行，因此`later()`函数的内容会在之后的一个时间执行（从现在起1000毫秒后）.

任何时候把一段代码包裹到`function`，并指定到对应事件（定时器、鼠标单击、Ajax请求等）执行，你就创建了一个*以后*代码块，从而在程序中引入异步。

### 异步 Console（Async Console）

没有规范或要求定义`console.*`方法是如何运行的--它们并不是JavaScript的官方部分，而是主机环境（译者注：如浏览器，服务端等）加入到JS当中的。

因此，不同的浏览器和JS环境随意实现，有时可能会导致令人困惑的行为。

尤其是有些浏览器和环境中，`console.log(...)`并不会立即输出给定的值。主要原因可能是I/O在许多程序中（不仅仅是JS）是一个非常缓慢和阻塞性的部分。因此，对浏览器来说，在后台异步处理`console`I/O可能性能更好（从页面/UI角度），你甚至不知道它的发生。

尽管不太常见，但可能看到这个场景（不是从代码层面，而是从外部（译者注：执行结果））：

```javascript
var a = {
    index: 1
};

// later
console.log( a ); // ??

// even later
a.index++;
```

我们通常认为`a`对象在`console.log(...)`语句执行时是个快照，输出比如`{index:1}`，以至于当`a.index++`执行的时候，只是严格地在输出   之后修改`a`。

大多数时候，前面的代码在你的开发工具中的控制台能够产生你期望的结果。但同样的代码也可能在觉得需要推迟`console`I/O到后台的浏览器中运行，那样的话，在浏览器控制台中，显示出`a`的时候，`a.index++`可能已经执行了，从而输出`{index:2}`。

到底在什么情况下`console`I/O会被推迟执行，或者是否能观测到这一现象是不确定的。记住，每当你调试一个在`console.log(...)`语句后修改对象的代码时，注意可能的异步I/O，你可能看到不可思议的结果出现。

**注意：** 如果你碰到这种很少见的情况，最好的选择时采用JS调试器的断点，而不是依赖`console`输出。另一个较好的方法是将查询的对象序列化为字符串，比如`JSON.stringify(...)`。

### 事件轮询（Event Loop）

让我们小小抱怨一下：尽管允许异步JS代码（正如我们刚刚看到的timeout一样），直到最近（ES6），JS本身从来没有内建的直接表示异步的概念。

**纳尼!?**似乎很疯狂对不对，事实上是真的。当被要求的时候，JS引擎除了在任何给定时刻执行程序中的单个代码块以外，什么都没做。

"谁要求的？"这是关键所在！

JS引擎并不是孤立运行的，它处在主机环境中，对绝大多数开发者来说是普通的web浏览器。在过去的几年里，JS已经从浏览器扩展到其它环境，比如服务端，通过Node.js。事实上，如今JS已经被嵌入到各种设备当中，从机器人到灯泡。

但在所有这些环境中，有一个通用的“线程”，采用该机制按时间来处理执行多个代码块，在每个时刻激活JS引擎，这称作“事件轮询”。

换句话说，JS引擎天生就没有时间的概念，但是有对任何JS代码片段按需执行的环境。正是这个环境调度“事件”（JS代码执行）。

因此，比如当你的JS代码发出了一个Ajax请求从服务端获取数据，你在一个函数（通常称为“回调”）中设置了一个“响应”代码，然后JS引擎告诉主机环境，“嗨，我现在准备中止执行了，但是当你完成网络请求，并且有数据了，请调用后面的这个函数”。

然后浏览器对这个网络响应设置监听，当有东西返回的时候，浏览器通过把回调函数插入事件轮询执行回调。

那么事件轮询是什么？

让我们通过一些伪代码将它概念化：

```javascript
// `eventLoop` is an array that acts as a queue (first-in, first-out)
var eventLoop = [ ];
var event;

// keep going "forever"
while (true) {
    // perform a "tick"
    if (eventLoop.length > 0) {
        // get the next event in the queue
        event = eventLoop.shift();

        // now, execute the next event
        try {
            event();
        }
        catch (err) {
            reportError(err);
        }
    }
}
```

当然，这只是用来说明这一概念的简化的伪代码，但是帮助我们理解的话，已经足够了。

如你所见，有一个如`while`循环所示的一个持续运行的循环，每次循环遍历称为一个“tick”。对每个tick来说，如果队列中有一个事件正在等待，则离开队列立即执行。这些事件就是你的回调函数。

需要注意的一点是，`setTimeout(...)`并不会把你的回调函数放入事件轮询队列，它干的只是设置一个定时器，当定时器过期时，主机环境把你的回调函数放入事件轮询，这样某个未来的tick就会拾起并执行这个回调函数。

假如那时事件队列中已经有20项了呢？你的回调函数得等，排队站在其它的后面。通常没有办法抢占队列以跳到队列前头。这就解释了为什么`setTimeout(...)`定时器可能没恰好在设定的时间触发。（通常来说）执行环境能够确保你的回调不会在你设定的时间前触发，但是可能在你设定的或者之后的时间触发，依事件队列的状态而定。

因此，换句话说，你的程序通常被分为几小块，在事件队列中一个接着一个的执行。从技术上讲，和你的程序没有直接关系的事件也能够在队列中插入。

**注意：** 我们之前提到的ES6改变了事件轮询队列的管理方法。它是一个正式的术语，但ES6指定了事件队列如何运行，从技术角度讲，它被纳入了JS引擎的范畴，而不仅仅是主机环境。这一改变的主要原因是ES6 Promises的引入，我们会在第三章讨论。因为他们需要能够对事件轮询队列的调度作直接、精细地控制（可见“协作”小节中`setTimeout(...0)`的讨论）

### 多线程（Parallel Threading）

把“异步”和“并行”理解为一个东西的想法很常见，但它们其实一点都不同。记住，异步是关于*现在* 和 *以后* 的界限。但并行是允许事情能够同时发生。

最常用的并行计算工具是多进程和多线程。多进程和多线程可以独立并且可能同时运行：单独的处理器或者单独的电脑，但多线程可共享单个进程的内存。

相反，事件轮询把工作分为多个任务并按序执行，不允许对共享内存的并行访问和修改。并行化和“序列化”可以在不同的线程中以协作的事件轮询形式共存。

多线程执行的插入和异步事件的插入发生的粒度不同。

例如：

```javascript
function later() {
    answer = answer * 2;
    console.log( "Meaning of life:", answer );
}
```

尽管`later()`的内容被视作单个事件轮询入口，当考虑到这段代码执行所在的线程时，事实上可能有许多低等级的操作。比如，`answer = answer * 2`需要首先加载`answer`的当前值，然后把`2`放在某个地方，然后执行乘法操作，获得结果并把它存回`answer`变量中。

在单线程环境中，线程队列中的任务项是低等级操作并不重要，因为无法中断线程。但如果是在并行系统中，两个不同的线程运行同一个程序，你可能得到预料不到的结果。

考虑如下代码：

```javascript
var a = 20;

function foo() {
    a = a + 1;
}

function bar() {
    a = a * 2;
}

// ajax(..) is some arbitrary Ajax function given by a library
ajax( "http://some.url.1", foo );
ajax( "http://some.url.2", bar );
```

在JS单线程下，如果`foo()`在`bar()`之前运行，结果就是`a`为42，但是如果`bar()`在`foo()`之前运行，则`a`为41。

然而，如果共享相同数据的JS事件并行执行，问题可能更难以捉摸。考虑如下两个伪代码任务，两个线程可以各自运行`foo()`和`bar()`代码，如果它们同时运行，考虑下会发生什么：

线程1:（`x`和`y`是临时内存位置）

```javascript
foo():
  a. load value of `a` in `X`
  b. store `1` in `Y`
  c. add `X` and `Y`, store result in `X`
  d. store value of `X` in `a`
```

线程2:（`x`和`y`是临时内存位置）

```javascript
bar():
  a. load value of `a` in `X`
  b. store `2` in `Y`
  c. multiply `X` and `Y`, store result in `X`
  d. store value of `X` in `a`
```

现在，我们假设两个线程是真正并行运行的。你可能注意到问题了，是吧？它们在临时步骤中使用了共享内存位置`x`和`y`。

如果按如下步骤执行下来，`a`是多少？

```javascript
1a  (load value of `a` in `X`   ==> `20`)
2a  (load value of `a` in `X`   ==> `20`)
1b  (store `1` in `Y`   ==> `1`)
2b  (store `2` in `Y`   ==> `2`)
1c  (add `X` and `Y`, store result in `X`   ==> `22`)
1d  (store value of `X` in `a`   ==> `22`)
2c  (multiply `X` and `Y`, store result in `X`   ==> `44`)
2d  (store value of `X` in `a`   ==> `44`)
```

结果`a`为`44`，但是这个顺序呢？

```javascript
1a  (load value of `a` in `X`   ==> `20`)
2a  (load value of `a` in `X`   ==> `20`)
2b  (store `2` in `Y`   ==> `2`)
1b  (store `1` in `Y`   ==> `1`)
2c  (multiply `X` and `Y`, store result in `X`   ==> `20`)
1c  (add `X` and `Y`, store result in `X`   ==> `21`)
1d  (store value of `X` in `a`   ==> `21`)
2d  (store value of `X` in `a`   ==> `21`)
```
最终`a`为`21`。

因此，线程编程非常具有欺骗性。如果你不采取特别措施来阻止这种相互干扰/插入，可能会得到非常奇怪并且不确定的结果，着实让人头疼。

JS从来不在线程间共享数据，这意味着不确定性不是大问题。但那并不是说JS总是确定性的。记得早些时候的`foo()`和`bar()`的相对顺序导致的不同结果吗(`41`还是`42`)？

**注意：** 尽管不明显，但并不是所有的不确定性都是坏的。有时是不相关的，有时是特意的。在下文以及后面几章中，我们会看到更多示例。

### 运行直至结束（Run-to-Completion）

因为JS是单线程的，`foo()`（和`bar()`）中的代码是原子性的，意思是一旦`foo()`开始运行，它的所有代码将会在`bar()`中任何代码执行前执行完，反之亦是如此。这就叫做“运行直至结束”。

事实上，如果`foo()`和`bar()`的内部有更多代码时，run-to-completion语义更明显，比如：

```javascript
var a = 1;
var b = 2;

function foo() {
    a++;
    b = b * a;
    a = b + 3;
}

function bar() {
    b--;
    a = 8 + b;
    b = a * 2;
}

// ajax(..) is some arbitrary Ajax function given by a library
ajax( "http://some.url.1", foo );
ajax( "http://some.url.2", bar );
```
因为`bar()`无法中断`foo()`，并且`foo()`也无法中断`bar()`,这段代码只有两个可能的结果，取决于哪一个首先运行--假如有多线程的话，`foo()`和`bar()`中的语句就可能相互交叉，可能的结果就会急剧增加！

块1是同步的（*现在* 发生），但是块2和3是异步的（*以后* 发生），这意味着它们的执行会被一个时间界限分隔开。

块1：

```javascript
var a = 1;
var b = 2;
```

块2（`foo()`）:

```javascript
a++;
b = b * a;
a = b + 3;
```

块3（`bar()`）：

```javascript
b--;
a = 8 + b;
b = a * 2;
```

块2和3都有可能第一个运行，因此这段程序可能有两个结果，如下：

结果1：

```javascript
var a = 1;
var b = 2;

// foo()
a++;
b = b * a;
a = b + 3;

// bar()
b--;
a = 8 + b;
b = a * 2;

a; // 11
b; // 22
```



结果2：

```javascript
var a = 1;
var b = 2;

// bar()
b--;
a = 8 + b;
b = a * 2;

// foo()
a++;
b = b * a;
a = b + 3;

a; // 183
b; // 180
```

同一段代码有两种结果表明仍然有不确定性，但这是在函数（事件）级的顺序问题，不是在多线程时语句级（或者说是表达式操作级）的顺序问题。换句话说，这比多线程更具确定性。

JS中这种函数顺序的非确定性行为通常称为“竞态”(race condition)，`foo()`和`bar()`会相互竞争，看谁先运行完。特别地，之所以称为“竞态”，是因为你无法可靠地预测`a`和`b`的结果会是怎样。

**注意：** 如果JS中有一个函数，它并不是运行直至结束的（run-to-completion），我们可能得到更多的结果，对吗？ES6确实引入了这么个东西（详见第四章“生成器”），但别担心，我们回头会讲的！


### 并发（Concurrency）

想象一个显示状态更新的列表的站点（比如一个社交网络的消息提示），当用户向下滚动列表的时候，网站逐步加载内容。为了实现这一效果，（至少）需要同时（即在同一个时间窗口，并不一定要同一时刻）分别执行两个“进程”。

**注意：** 此处我们在“进程”上使用了引号，因为这并不是计算机科学中的操作系统级的进程概念。这是虚拟进程，或者叫任务，代表了一个逻辑连接，序列化操作。相比于“任务”，我们更喜欢用“进程”，因为从术语层面来说，这更匹配我们所讨论的概念定义。

当用户向下滚动页面以获取新内容时，第一个“进程”负责`onscroll`事件（发Ajax请求获取新内容）。第二个“进程”负责接收Ajax响应（把内容渲染到页面上）。

很明显，如果用户快速滚动，在获取第一个响应并处理的时候，你会发现两个甚至更多的`onscroll`事件触发。因此，你会发现`onscroll`事件和Ajax响应事件相互交叉，快速地触发。

并发是指两个或多个“进程”在同一时间段内同时执行，不管它们各自的操作是否是并行的（同一时刻在不同的处理器或者核心上）。相比于操作级并行（不同的处理器线程），你可以认为并发是“进程”级（或任务级）的并行。

**注意：** 并发同样引入了一个可选的“多进程”交互的概念，之后会有所提及。

对于一个给定的时间窗口（几秒钟，大概为用户的滚动时间），让我们将每个独立的“进程”想象为一系列事件/操作：

“进程”1（`onscroll`事件）：

```javascript
onscroll, request 1
onscroll, request 2
onscroll, request 3
onscroll, request 4
onscroll, request 5
onscroll, request 6
onscroll, request 7
```

“进程”2（Ajax响应事件）：

```javascript
response 1
response 2
response 3
response 4
response 5
response 6
response 7
```

很可能一个`onscroll`时间和一个Ajax响应事件同时得到处理。例如，从时间轴角度想象一下这些事件：

```javascript
onscroll, request 1
onscroll, request 2          response 1
onscroll, request 3          response 2
response 3
onscroll, request 4
onscroll, request 5
onscroll, request 6          response 4
onscroll, request 7
response 6
response 5
response 7
```

但是，请回想一下本章前面部分提到的概念，JS在同一时刻只能处理一个事件，因此，要么`onscroll, request 2`先发生，要么`response 1`先发生，它们不可能在同一时刻发生。就像一个学校自助餐厅的孩子们一样，不论在门外挤成什么样，他们都不得不排成队来吃午餐!

让我们把所有这些事件想象为事件轮询队列。

事件轮询队列：

```javascript
onscroll, request 1   <--- Process 1 starts
onscroll, request 2
response 1            <--- Process 2 starts
onscroll, request 3
response 2
response 3
onscroll, request 4
onscroll, request 5
onscroll, request 6
response 4
onscroll, request 7   <--- Process 1 finishes
response 6
response 5
response 7            <--- Process 2 finishes
```

“进程1“和”进程2“并发运行（任务级并行），但它们各自的事件在事件队列中序列化运行。

顺便提一句，注意到`response 6`和`response 5`是怎么没有按照预期顺序返回的吗？

单线程事件轮询是并发的一种表现形式（当然还有其它形式，后面会提及）。

### 非交互（Noninteracting）

尽管在同一个程序中，两个或更多的”进程“并发地插入各自的步骤/事件，但如果任务不相关，它们没必要交互。**如果它们不交互，非确定性完全可以接受**。

例如：

```javascript
var res = {};

function foo(results) {
    res.foo = results;
}

function bar(results) {
    res.bar = results;
}

// ajax(..) is some arbitrary Ajax function given by a library
ajax( "http://some.url.1", foo );
ajax( "http://some.url.2", bar );
```

`foo()`和`bar()`是两个并发”进程“，无法确定哪一个先触发。但我们构建的这个程序，不管哪个先触发都没关系，因为它们是相互独立的，不需要交互。

这不是个”竞态“bug，因为不论顺序如何，代码都能正常工作。

### 交互（Interaction）

更多情况下，间接地通过作用域和/或DOM，并发”进程“有必要交互。当发生这种交互的时候，你需要协调这些交互来阻止之前提到的”竞态“。

以下是一个由于潜在顺序问题，两个并发”进程“交互的简单例子，有时可能导致程序异常：

```javascript
var res = [];

function response(data) {
    res.push( data );
}

// ajax(..) is some arbitrary Ajax function given by a library
ajax( "http://some.url.1", response );
ajax( "http://some.url.2", response );
```

并发”进程“是两个`response()`调用，用来处理Ajax响应。它们谁都可能先发生。

我们假定预期的结果是`res[0]`保存了`http://some.url.1`调用返回的结果，`res[1]`保存了`http://some.url.2`调用返回的结果。有时是这样，但有时结果完全反过来，依谁先完成而定。这一不确定性很像一个”竞态“bug。

**注意：** 在类似这种情形下，作出假设前一定要谨慎。例如，或许根据处理的任务（比如数据库任务或者其它抓取静态文件任务），开发人员注意到`http://some.url.2`的响应”总是“比`http://some.url.1`慢，因此，观测的结果总是和预期的顺序一致，这很常见。即使两个请求发向了同一个服务器，并且服务器按照特定顺序返回了响应，也不能保证返回浏览器的响应的顺序。

因此，为了处理这种竞态，你可以协调下有序交互：

```javascript
var res = [];

function response(data) {
    if (data.url == "http://some.url.1") {
        res[0] = data;
    }
    else if (data.url == "http://some.url.2") {
        res[1] = data;
    }
}

// ajax(..) is some arbitrary Ajax function given by a library
ajax( "http://some.url.1", response );
ajax( "http://some.url.2", response );
```

不管哪个Ajax响应先返回，我们检查`data.url`（当然，可以假定是从服务器端返回的），确定响应数据应该占据`res`数组的哪个位置。`res[0]`永远存放着`http://some.url.1`的结果，`res[1]`永远存放着`http://some.url.2`的结果。通过简单的协调，我们消除了竞态不确定性。

同样的道理，如果多个并发函数通过共享DOM交互调用，比如，其中一个更新`<div>`的内容，其它的更新`<div>`的样式或者属性（比如，一旦有内容，则让DOM元素可见）。你可能不想在有内容前显示DOM元素，因此协调必须确保合适的交互顺序。

没有协调交互的话，一些并发场景总是会导致异常（不只是有时）。考虑如下：

```javascript
var a, b;

function foo(x) {
    a = x * 2;
    baz();
}

function bar(y) {
    b = y * 2;
    baz();
}

function baz() {
    console.log(a + b);
}

// ajax(..) is some arbitrary Ajax function given by a library
ajax( "http://some.url.1", foo );
ajax( "http://some.url.2", bar );
```

在这个例子当中，不论`foo()`和`bar()`谁先触发，总会导致`baz()`过早运行（`a`或者`b`总有一个是`undefined`）,但在第二次调用`baz()`时就正常了，因为`a`和`b`都有值了。

有不同方法可以处理这种情况，以下是个简单的方式：

```javascript
var a, b;

function foo(x) {
    a = x * 2;
    if (a && b) {
        baz();
    }
}

function bar(y) {
    b = y * 2;
    if (a && b) {
        baz();
    }
}

function baz() {
    console.log( a + b );
}

// ajax(..) is some arbitrary Ajax function given by a library
ajax( "http://some.url.1", foo );
ajax( "http://some.url.2", bar );
```

`baz()`外围的`if (a && b)`条件一般称为”门（gate）“，因为不确定`a`和`b`的到达顺序，但可以等到门打开（调用`baz()`）时，两个都到达。

另一个可能碰到的并发交互情形有时称为”竞争（race）“，更确切地应该叫”闩（latch）“。它的特点是”只有第一个赢“。此时，非确定性是可以接受的，你明确地说明了”竞争“直至终点，只产生一个胜者是OK的。

考虑如下异常代码：

```javascript
var a;

function foo(x) {
    a = x * 2;
    baz();
}

function bar(x) {
    a = x / 2;
    baz();
}

function baz() {
    console.log( a );
}

// ajax(..) is some arbitrary Ajax function given by a library
ajax( "http://some.url.1", foo );
ajax( "http://some.url.2", bar );
```

无论哪一个（`foo()`或者`bar()`）后触发，不仅会重写另一个对`a`的赋值，而且会重复调用`baz()`（这是我们不希望看到的）。

因此，我们可以采用一个简单的闩来协调交互，只让第一个通过：

```javascript
var a;

function foo(x) {
    if (a == undefined) {
        a = x * 2;
        baz();
    }
}

function bar(x) {
    if (a == undefined) {
        a = x / 2;
        baz();
    }
}

function baz() {
    console.log( a );
}

// ajax(..) is some arbitrary Ajax function given by a library
ajax( "http://some.url.1", foo );
ajax( "http://some.url.2", bar );
```

`if (a == undefined)`条件语句只允许`foo()`和`bar()`中的第一个通过，第二个（以及其后所有的）调用会被忽略。

**注意：** 在所有这些场景中，为便于说明，我们一直采用全局变量，但并没有理由一定要这样做。只要相关的函数能够访问变量（通过作用域），同样能运行的很好。依赖词法作用域变量，即此处的全局变量，是这些并发协调形式的一个明显缺点。随着我们深入之后的几章，会看到其它一些更简洁的形式。

### 协作（Cooperation）

另一种并发协调的表现形式是”并发协作“。此处的关注点不是通过作用域的值共享进行交互（尽管仍然允许！），而是运行一个长时的”进程“，并将其分成几步或几个批次，使得其它的并发”进程“有机会把他们的操作插入到事件轮询队列中。

例如,假设有一个AJax响应处理函数，需要遍历一长串结果以改变值。此处为保持代码简洁，我们采用`Array#map(..)`：

```javascript
var res = [];

// `response(..)` receives array of results from the Ajax call
function response(data) {
    // add onto existing `res` array
    res = res.concat(
        // make a new transformed array with all `data` values doubled
        data.map( function(val){
            return val * 2;
        } )
    );
}

// ajax(..) is some arbitrary Ajax function given by a library
ajax( "http://some.url.1", response );
ajax( "http://some.url.2", response );
```

如果`"http://some.url.1"`首先返回结果，整个结果会马上映射到`res`中去。如果只是几千或者更少的数据，这通常不是个大问题。但如果有1000万条数据，可能需要花上一段时间（一台性能强劲的笔记本可能要几秒钟，移动设备就更长了）。

当这样一个”进程“运行的时候，页面上的其它东西都会停止，包括其它`response(..)`调用、UI更新，更不用说像滚动、输入、按钮点击以及诸如此类的用户事件了。那真痛苦。

因此，为了让并发系统协作性更好，可以采用一种更友好并且不会霸占事件轮询队列的方式，你可以采用异步批次的方式处理这些数据。在每个批次被推入事件队列后，让其它批次等待事件发生。

以下是一个非常简单的方法;

```javascript
var res = [];

// `response(..)` receives array of results from the Ajax call
function response(data) {
    // let's just do 1000 at a time
    var chunk = data.splice( 0, 1000 );

    // add onto existing `res` array
    res = res.concat(
        // make a new transformed array with all `chunk` values doubled
        chunk.map( function(val){
            return val * 2;
        } )
    );

    // anything left to process?
    if (data.length > 0) {
        // async schedule next batch
        setTimeout( function(){
            response( data );
        }, 0 );
    }
}

// ajax(..) is some arbitrary Ajax function given by a library
ajax( "http://some.url.1", response );
ajax( "http://some.url.2", response );
```

我们把数据集分成每个最大1000项的数据块进行处理。这样，我们能够确保一个短时的”进程“，即使这意味着有更多的”进程“随之而来。将这些”进程“插入到事件轮询队列能够让我的网站/应用响应性更好，性能更佳。

当然，我们没有协调交互来控制这些”进程“的顺序，因此结果的顺序也是不可预测的。如果顺序是必须的，你可以采用之前讨论的交互技术，或者下一章中我们会讲的技术。

我们采用`setTimeout(..0)`(hack)进行异步调度，这仅仅意味着”把这个函数插入到事件轮询队列的末尾“。

**注意：** 技术上来说，`setTimeout(..0)`并不会直接把任务项插入到事件轮询队列中，定时器会在下次机会中插入事件。例如，两个先后`setTimeout(..0)`的调用并不能严格保证按照调用顺序得到处理。所以，可能看到各种各样的情形，比如定时器漂移，这样事件的顺序是不可预测的。在Node.js中，有一个类似的方法叫`process.nextTick(..)`。尽管很方便（通常性能也更好），但没有一个直接、横跨所有环境的确保异步事件顺序的方式（至少目前如此）。在下一节中，我们会讨论更多关于这个话题的细节。

### 作业（Jobs）

ES6中，除了事件队列以外，还有一个新概念，叫”作业队列“（Job queue）。你最有可能在Promises的异步行为中（详见第三章）接触到这一概念。

不幸的是，此时，它只是一个没有外露接口的机制，因此，证明它可能有点复杂。因此，我们只从概念上描述它，以便当我们在第三章讨论Promises的异步行为时，你能够理解这些操作是怎样调度和处理的。

因此，我认为对”作业队列“理解的最好方式是，它是一个挂载在事件轮询队列中每个tick的末尾的队列。某些可能在事件队列的tick期间发生的潜在异步操作并不会导致一个完整的新事件插入到事件轮询队列中，而是在当前tick的作业队列末尾加入一个项目（即作业）。

就好像说：”哦，我这儿有个需要之后处理的其它东西，但确保在其它事情发生之前立即发生“。

或者，打个比方：事件队列就像游乐场的过山车，一旦你玩过一次，你必须到队尾去排队。而作业队列就像一旦你玩结束了，之后插队又继续玩了。

一个作业可能引发更多的作业插入到同一队列的末尾。因此，从理论上来说，一个作业”循环“（一个作业不断地加入另一个作业）很可能无限延伸，因而导致程序无法移到下一个事件循环tick中。从概念上来说，这跟一个长时运行或者无限循环（比如`while(true)...`）几乎一模一样。

作业有点像`setTimeout(..0)`hack，但是能够更好地定义和确保顺序：**以后，但尽可能快点**。

假设一个调度Jobs（直接的，没有采用hacks）的API，叫做`schedule(..)`，如下：

```javascript
console.log( "A" );

setTimeout( function(){
    console.log( "B" );
}, 0 );

// theoretical "Job API"
schedule( function(){
    console.log( "C" );

    schedule( function(){
        console.log( "D" );
    } );
} );
```

你可能认为会输出`A B C D`，但其实是输出`A C D B`，因为作业是在当前事件轮询tick的末尾发生的，而定时器触发时，是调度到下一个事件轮询tick的（如果存在的话）。

在第三章，我们将会看到，Promises的异步行为是基于Jobs的，因此有必要搞清楚它和事件轮询的关系。

### 语句排序（Statement Ordering）

我们代码中写的语句顺序并不一定和JS引擎执行的顺序一致。这一论断看起来似乎有点奇怪，那么我们简单地看下。

但在开始之前，我们应该搞清楚一些事情：从程序角度而言，语言的规则/语法为语句排序决定了代码的行为是可预测的和可信赖的。因此，我们所讨论的**不是你在JS程序中能观察到的东西**。

**警告：** 如果你能观察到编译器语句重排，就像我们将要举例说明的那样，这很明显违背了规范的要求，毫无疑问会是JS引擎的一个大bug--一个应该立即报告并修复的bug！但是当那就是你程序中的一个bug时（比如一个”竞态“），你怀疑JS引擎发生了一些不可思议的事情，这非常普遍--因此，首先看看是不是重排导致的问题，仔细看。采用JS调试器，设置断点，一行行单步跟踪代码，会成为你揪出代码中这一类bug最强有力的工具。

如下：

```javascript
var a, b;

a = 10;
b = 30;

a = a + 1;
b = b + 1;

console.log( a + b ); // 42
```

这段代码没有异步操作（除了之前提到的`console`异步I/O），因此最有可能的假设是它会从头到尾一行行地执行。

但很可能在JS引擎编译完这段代码后（是的，JS会编译），发现重排（安全地）一下代码可能让执行速度更快些。本质而言，只要你看不到这种重排，任何事情都是焦点（译者注：大概意思是关注点就不在重排上了）。

例如，引擎发现像这样执行代码也许更快些：

```javascript
var a, b;

a = 10;
a++;

b = 30;
b++;

console.log( a + b ); // 42
```

或者这样：

```javascript
var a, b;

a = 11;
b = 31;

console.log( a + b ); // 42
```

甚至：

```javascript
// because `a` and `b` aren't used anymore, we can
// inline and don't even need them!
console.log( 42 ); // 42
```

在以上所有情况当中，JS引擎在编译阶段进行安全的优化，最终的结果一样。

但是有一种情况下，这种特定的优化坑能不安全，因而不允许这样（当然，并不是说一点都不优化）：

```javascript
var a, b;

a = 10;
b = 30;

// we need `a` and `b` in their preincremented state!
console.log( a * b ); // 300

a = a + 1;
b = b + 1;

console.log( a + b ); // 42
```

其它因编译器重排导致副作用的例子（因此也必须禁止）可能包括一些像具有副作用的函数调用（尤其是getter函数），或者ES6的Proxy对象。

考虑如下代码：

```javascript
function foo() {
    console.log( b );
    return 1;
}

var a, b, c;

// ES5.1 getter literal syntax
c = {
    get bar() {
        console.log( a );
        return 1;
    }
};

a = 10;
b = 30;

a += foo();             // 30
b += c.bar;             // 11

console.log( a + b );   // 42
```

要不是因为`console.log(...)`语句（仅是为了说明这种可观测的副作用而采用的简便形式），如果乐意（谁知道它是否会呢），JS引擎早把代码重排成这样了：

```javascript
// ...

a = 10 + foo();
b = 30 + c.bar;

// ...
```

谢天谢地，尽管JS语法能够让我们免于遭受编译器语句重排带来的噩梦困扰，但理解编写的源码（至上而下的方式）和实际编译后执行的方式之间的联系有多薄弱是十分重要的。

编译器语句重排是并发和交互（interaction）的微隐喻。作为一个普遍概念，这一意识能够帮助你更好地理解JS代码的异步流问题。

### 回顾

日常开发中，一个JS程序总是被分成两个或更多的块，第一个块*现在*执行，下一个块*以后*执行，对应一个事件。尽管程序是按块执行的，但所有块共享对程序的作用域和状态的访问，因此，每次对状态的修改都是在前一个状态的基础上的。

只要有事件要运行，事件轮询会一直运行直至队列为空。对事件轮询的每次迭代称为一个”tick“。用户交互、IO和定时器都会在事件队列中进行事件排队。

并发是两条或多条事件链随着时间相互交叉，以至于从更高层面来看，他们似乎是同时运行的（即使在任何给定时刻，只有一个事件被处理）。

比如，为了确保顺序或者为了阻止”竞态“，通常很有必要在这些并发的”进程“（有别于操作系统级的进程概念）间进行某种形式的交互协调。也可以通过把这些”进程“拆分成多个小块，从而允许其它”进程“插入进来。




