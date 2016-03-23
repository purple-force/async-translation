## 你不知道JS：异步

## 第三章：Promises

在第二章，我们指出了采用回调来表达异步和管理并发时的两种主要不足：缺乏序列化和可信度。既然我们对这些问题更熟悉了，是时候将注意力集中到解决这些问题的模式上了。

我们想要处理的第一个问题是*控制权反转*，信任很难维持，脆弱且容易丢失。

回想一下我们把程序的*延续*包裹在回调函数中，把回调交给其它方（甚至可能是外部代码），然后祈祷能过正确地调用回调函数。

我们这样做的原因是我们想说，“这个是在当前步骤结束之后，*后来*发生的”。

但要是我们能够反逆转*控制权反转*呢？要是能够让第三方返回一个结果，让我们知道它的任务结束时间，然后我们的代码决定接下来该做什么呢？而不是把程序的延续交给第三方。

这一范例称为**Promises**。

随着诸如开发人员和规范制定人员之类的人不顾一切地寻求解决代码/设计中的回调地狱错乱问题的方法，Promises将会撼动整个JS世界。事实上，JS/DOM中新加入的API大多数都是建立在Promises上的。那么深入学习Promises可能是个不错的注意，不是吗？

**注意：** 本章中”立刻（immediately）“一词将会反复提及，通常指某些Promise的解析行为。然而，从本质上来说，”立刻“一词是根据作业队列（Job queue）（见第一章）来说的，不是*现在（now）*层面上严格同步的意思。

## Promise是什么？（What Is a Promise?）

当开发人员学习一种新的技术或者模式时，第一步通常是”给我看代码！“对我们而言，直接进入（译者注：指直接看代码）并学习很自然。

但结果是，只关注API的话，就会丢失一些抽象概念。Promise是这样的一种工具，如果不理解它是做什么的，只是学会使用API的话，使用起来会很痛苦。

因此，在我展示Promise代码之前，我想从概念上完完整整地解释Promise到底是什么。我希望这能够指导你更好地将Promise理论整合到自己的异步流中。

明确这点，让我们看一下Promise是什么的两个不同类比。

### 未来值（Future Value）

假设这个场景：我走到快餐店的柜台，点了一个奶酪汉堡，我递给收银员$1.47的零钱。通过下单和支付，我已经作了一次请求，希望有*值*（奶酪汉堡）返回。我已经开始了一次交易。

但通常，奶酪汉堡并不是立刻就能给我。收银员给了我一个代替奶酪汉堡的东西：一个有订单号的收据。这个订单号是一个IOU（”i owe you
“）的*承诺*，确保最后我能拿到我的奶酪汉堡。

我拿着我的收据和订单号。我知道它代表着我*未来的奶酪汉堡*，因此我再也不用担心了--除了我很饿之外！

当我在等的时候，我还可以做其它事，比如给我的朋友发短信说，”嗨，你能和我一起吃午餐吗？我要吃个奶酪汉堡。“

即使还没拿到手，我已经开始推演我的*未来的奶酪汉堡*。我的大脑之所以能这样做是因为它把订单号当成了奶酪汉堡占位符。本质上这个占位符使得值*与时间无关*。它是**未来值**。

最终，我听到了，”113号！“我很高兴地拿着订单走向柜台。把订单递给收银员后，我拿到了我的奶酪汉堡。

换句话说，一旦我的*未来值*准备好了，我就可以拿承诺值（value-promise）换真正的值（value）了。

但还可能有另一种结果。他们叫了我的订单号，但是当我去取奶酪汉堡时，收银员很遗憾地告诉我，”很抱歉，我们恰巧卖完了奶酪汉堡“。抛开客户沮丧的场景来看，我们可以看到*未来值*的一个重要特点：它们既可能于是着成功，也可能预示着失败。

每次我点一个奶酪汉堡，我都知道可能最终我会得到一个，或者很遗憾地听到奶酪汉堡卖完了，不得不另寻其它作为午餐了。

**注意：** 在代码中，事情并不是如此简单，因为比方来说，订单号可能永远都不会被叫到，此时我们就无限地处在了悬而未决的状态。我们稍后处理这种情形。

### 现在值和以后值（Values Now and Later）

如果应用到代码中，听起来似乎太抽象了。让我们更具体一点。

然而，在我们以这种方式引入Promise的工作原理前，我们将从已知的代码中--回调--去探究如何处理这些*未来值*的。

当你写代码来推演一个值的时候，比如对一个`number`进行数学运算。不论是否意识到，关于那个值，你已经假定了一些重要的东西，即它已经是一个具体的*现在*值:

```javascript
var x, y = 2;

console.log( x + y ); // NaN  <-- because `x` isn't set yet
```

`x+y`的操作假定`x`和`y`已经设置好值了。简而言之，我们假定`x`和`y`已经被*解析*了（resolved）。

期望`+`操作符本身能够检测和等到`x`和`y`都解析完（即准备好，然后去进行`+`的操作）显然没什么意义。如果不同的语句*现在*完成，其它的*以后*完成，这样会使得程序混乱，对吗?

如果一个语句（或者两个都）还没有完成，你该如何推演它们之间的关系呢？如果语句2依赖语句1的完成，只会有两个结果：要么语句1立马完成（right *now*），一切OK，要么语句1还没完成，进而语句2也将失败。

如果觉得听起来有点耳熟，很好！

让我们回到`x+y`的数学操作。想象以一种方式说，”将`x`和`y`相加，如果其中任一个还没准备好，只需等等直至二者都准备好了。尽可能快的将二者相加。“

你的脑袋可能立即跳到了回调。OK，那么...

```javascript
function add(getX,getY,cb) {
    var x, y;
    getX( function(xVal){
        x = xVal;
        // both are ready?
        if (y != undefined) {
            cb( x + y );    // send along sum
        }
    } );
    getY( function(yVal){
        y = yVal;
        // both are ready?
        if (x != undefined) {
            cb( x + y );    // send along sum
        }
    } );
}

// `fetchX()` and `fetchY()` are sync or async
// functions
add( fetchX, fetchY, function(sum){
    console.log( sum ); // that was easy, huh?
} );
```

花几分钟来完全领会这段代码的美（或不足）。

不可否认，尽管有点丑，但是这种异步模式中还是有值得注意的地方。

在那段代码中，我们把`x`和`y`视作未来值，并且表示操作的`add(..)`并不关心`x`和`y`是否都立刻准备好了。换句话说，它统一了*现在*和*以后*，如此我们就可以依赖`add(..)`操作生成的一个可预测的结果。

通过使用时序一致的`add(..)`---*现在*和*以后*的行为都保持一致--使得异步代码更容易推演了。

更通俗一点来说：为了统一处理*现在*和*以后*，我们把它们都变成了*以后*。所有的操作都变成异步的了。

当然，这种简单的基于回调不是太令人满意。推演未来值而不需要担心时间维度，即值是否可用，对于这一好处而言，这只是一小步。

### 承诺值（Promise Value）

本章中，我们肯定会探讨Promise的更多细节--所以，如果有点困惑，别担心--但让我们简单看看如何采用`Promise`来实现`x+y`的操作：

```javascript
function add(xPromise,yPromise) {
    // `Promise.all([ .. ])` takes an array of promises,
    // and returns a new promise that waits on them
    // all to finish
    return Promise.all( [xPromise, yPromise] )

    // when that promise is resolved, let's take the
    // received `X` and `Y` values and add them together.
    .then( function(values){
        // `values` is an array of the messages from the
        // previously resolved promises
        return values[0] + values[1];
    } );
}

// `fetchX()` and `fetchY()` return promises for
// their respective values, which may be ready
// *now* or *later*.
add( fetchX(), fetchY() )

// we get a promise back for the sum of those
// two numbers.
// now we chain-call `then(..)` to wait for the
// resolution of that returned promise.
.then( function(sum){
    console.log( sum ); // that was easier!
} );
```

这段代码中有两层Promise。

`fetchX()`和`fetchY()`是直接调用的，返回值（promises！）传入`add(..)`。这些promise代表的潜在值可能是*现在*或者*以后*准备好，但每个promise统一了这一行为。我们以跟时间无关的方式推演（reason about）`x`和`y`。他们是*未来值*。

第二层是`add(..)`创建并返回的promise（通过`Promise.all([ .. ])`），在此通过调用`then(..)`等待。当`add(..)`操作完成后，`sum`未来值就准备好了，之后我们就可以输出它了。我们隐藏了`add(..)`中等待`x`和`y`未来值的内部逻辑。

**注意：** 在`add(..)`内部，`Promise.all([ .. ])`创建了一个promise（等待`promiseX`和`promiseY`解析）。`.then(..)`的链式调用创建了一个promise，`return values[0] + values[1]`立即得到解析（相加的结果）。因此，`add(..)`调用之后的链式`then(..)`调用--代码末尾--实际上是对返回的第二个promise进行操作，而不是`Promise.all([ .. ])`创建的第一个。另外，尽管我们没有对第二个`then(..)`进行链式操作，它其实也生成了一个promise。这种Promise的链式调用会在本章后面作更详尽的说明。

就像奶酪汉堡订单一样，一个Promise的解析结果可能不是成功（fulfillment）而是失败（rejection）的。不像一个成功的promise，其值总是程序化的（programmatic，译者注：指通过程序设定），一个失败值--通常称为“失败原因”--既可通过程序逻辑直接设定，又可由隐式地运行异常导致。

通过Promise，`then(..) `调用实际能够接受两个函数，一个用作成功（fulfillment），第二个用作失败（rejection）：

```javascript
add( fetchX(), fetchY() )
.then(
    // fullfillment handler
    function(sum) {
        console.log( sum );
    },
    // rejection handler
    function(err) {
        console.error( err ); // bummer!
    }
);
```

如果获取`x`或者`y`出错，或者在相加过程中有什么出错了，则`add(..)`返回的promise被置为失败状态，传入`then(..)`的第二个回调错误处理函数将接受promise返回的失败值。

因为Promise从外面封装了跟时间无关的状态--等潜在值的成功或失败--Promise本身是跟时间无关的，因此Promise可以以可预测的方式组织（组合）起来，不管时间或结果。

此外，一旦Promise解析了，其状态永远不变--自那一时刻起，变成了*不可更改的值*--可以按需多次监听。

**注意：** 因为Promise一旦解析，就无法从外面更改其状态了，这样把值传递给第三方就很安全，因为无法无意或恶意地修改它。尤其是当有多个当事方监听这一个Promise的解析结果时更是如此。一方无法影响另一方对Promise解析的监听。不可变性听起来似乎像一个学术课题，但它其实是Promise设计中十分重要的方面之一，不能随意忽略。

那是理解Promise最强大和重要的概念之一。有一定实践经验后，你也能通过丑陋的回调组合来专门实现同样的效果，但那并不是一个真正有效地策略，尤其是你必须一次又一次地这么做。

Promise是一种很容易封装和构成*未来值*的可重复性机制。

### 完成事件（Completion Event）

正如我们刚刚看到的，一个Promise的行为就像一个*未来值*。但还可以采用另一种方式来思考Promise：在异步任务的两步或多步中--一个时序的this-then-that--作为一种流控制机制。

假设调用函数`foo(..)`来执行某个任务。我们不知道任何细节，也不关心。可能是立即完成的，也可能花一些时间。

我们只需简单地知道`foo(..)`什么时候完成，以便我们能够转到下一个任务中去。换句话说，我们想要通过某种方式来通知我们`foo(..)`已经完成了，以便于*继续*做其它事。

在传统JS中，如果你需要监听某个通知（notification），你可能会想到事件。因此，我们可以重构通知需求为需要监听`foo(..)`发出的*完成*（或者*继续*）事件。

**注意：** 称之为“完成事件”还是“继续事件”取决于你自己。关注点更应该放在`foo(..)`发生了什么或者`foo(..)`后会发生什么。这两个角度都是准确且有用的。事件通知告诉我们`foo(..)`已经完成了，同样也可以说是能够进行下一步了。事件通知时调用的传入的回调函数本身就是我们之前提到的*延续(continuation)*。因为*完成事件*更关注于`foo(..)`,此时我们也更关注于它。在本篇其余部分，会倾向于采用*完成事件*。

采用回调函数，“通知”就是任务（`foo(..)`）运行的回调。但采用Promise时，我们转变一下关系，希望我们可以从`foo(..)`中监听事件，当被通知的时候，进行相应的处理。

首先，考虑如下伪代码：

```javascript
foo(x) {
    // start doing something that could take a while
}

foo( 42 )

on (foo "completion") {
    // now we can do the next step!
}

on (foo "error") {
    // oops, something went wrong in `foo(..)`
}
```

我们调用`foo(..)`，然后设置了两个事件监听器，一个监听`"completion"`事件，一个监听`"error"`事件--`foo(..)`调用的两个可能的结果。本质上，`foo(..)`并不知道调用代码已经订阅了这些事件，很好的实现了关注点分离。

很不幸的是，这种代码需要JS环境的一些“魔法”，但并不存在（并且可能有点不切实际）。以下是我们在JS中以更自然的方式实现的代码：

```javascript
function foo(x) {
    // start doing something that could take a while

    // make a `listener` event notification
    // capability to return

    return listener;
}

var evt = foo( 42 );

evt.on( "completion", function(){
    // now we can do the next step!
} );

evt.on( "failure", function(err){
    // oops, something went wrong in `foo(..)`
} );
```

`foo(..)`显式地创建了事件订阅功能用作返回，然后调用代码接受它并
针对它注册了两个事件处理函数。

从正常的回调导向的代码反转（控制权），应该是很明显的，也是特意这么做的。`foo(..)`返回一个称之为`evt`的事件功能，用它来接收回调函数。

但是如果你回想一下第二章，回调函数本身代表*控制权反转（inversion of control）*。因此，反转回调模式通常称之为*反转的反转（inversion of inversion）*，或者*不反转控制权（uninversion of control）*--在我们需要以它为主的地方恢复调用代码的控制权。

这样做的一大好处就是代码的多个独立部分可以获得事件监听能力，当`foo(..)`完成的时候，它们可以独立地接到通知并进行后续步骤：

```javascript
var evt = foo( 42 );

// let `bar(..)` listen to `foo(..)`'s completion
bar( evt );

// also, let `baz(..)` listen to `foo(..)`'s completion
baz( evt );
```

*不反转控制权（Uninversion of control）*能够实现更好的关注点分离，`bar(..)`和`baz(..)`不需要关心`foo(..)`是如何调用的。同样的，`foo(..)`也不需要知道或者关心`bar(..)`和`baz(..)`的存在或者`foo(..)`完成时等着被通知。

本质上来说，这个`evt`对象是不同关注点之间的一个中立的第三方协商。

### Promise “事件”（Promise "Events"）

现在，你可能猜想，`evt`事件监听能力是对Promise的一个模拟。

在基于Promise的方式中，之前的代码会让`foo(..)`生成并返回一个`Promise`实例，那个promise之后会被传入`bar(..)`和`baz(..)`。

**注意：** 我们监听的Promise解析“事件”并不是严格意义上的事件（尽管某些地方看起来像事件的行为），并且通常也不称为`"completion"`或者`"error"`。反而，我们采用`then(..)`来注册一个`then(..)`事件。或者更准确一点，`then(..)`注册了`"fulfillment"`和/或`"rejection"`事件，尽管我们在代码中无法明确地看到这些东西。

考虑如下代码：

```javascript
function foo(x) {
    // start doing something that could take a while

    // construct and return a promise
    return new Promise( function(resolve,reject){
        // eventually, call `resolve(..)` or `reject(..)`,
        // which are the resolution callbacks for
        // the promise.
    } );
}

var p = foo( 42 );

bar( p );

baz( p );
```

**注意：** 采用`new Promise( function(..){ .. } )`的这种模式通常称为“[暴露构造函数（revealing constructor）](https://blog.domenic.me/the-revealing-constructor-pattern/)”(译者注：指Promise构造函数暴露了内部的功能，但只针对构造promise对象的代码)。传入的函数立即执行（不是像传入`then(..)`中的回调函数异步推迟执行的），该函数接收两个参数，此处我们命名为`resolve`和`reject`。这两个是promise的解析函数。`resolve(..)`通常标志着成功（fulfillment），`reject(..)`通常标志着失败（rejection）。

你可能在猜想`bar(..)`和`baz(..)`的内部是什么样子：

```javascript
function bar(fooPromise) {
    // listen for `foo(..)` to complete
    fooPromise.then(
        function(){
            // `foo(..)` has now finished, so
            // do `bar(..)`'s task
        },
        function(){
            // oops, something went wrong in `foo(..)`
        }
    );
}

// ditto for `baz(..)`
```

Promise解析并不一定像我们检查Promise并作为*未来值*时一样，需要随之传送一则信息。它可以简单地用作流控制信号，正如前段代码中用到的一样。

另一种实现方法是：

```javascript
function bar() {
    // `foo(..)` has definitely finished, so
    // do `bar(..)`'s task
}

function oopsBar() {
    // oops, something went wrong in `foo(..)`,
    // so `bar(..)` didn't run
}

// ditto for `baz()` and `oopsBaz()`

var p = foo( 42 );

p.then( bar, oopsBar );

p.then( baz, oopsBaz );
```

**注意：** 如果你之前已经看过基于Promise的代码，你可能认为最后两行代码可以采用链式方式写作`p.then( .. ).then( .. )`，而不是`p.then(..); p.then(..)`。注意，结果完全不同。现在看起来，差异还不是很明显，但它其实是一种不同于迄今为止我们见过的异步模式：分离/分叉(splitting/forking)。别担心！后面我们会讲。

我们用promise来控制`bar(..)`和`baz(..)`何时执行，而不是把`p` promise传入到`bar(..)`和`baz(..)`中。主要的差异在于错误处理。

在第一段代码中，不论`foo(..)`成功还是失败，`bar(..)`都会调用，如果被通知到`foo(..)`失败了，由`bar(..)`处理自己的回退逻辑。很明显，`baz(..)`也是一样。

在第二段代码中，只在`foo(..)`成功的时候调用`bar(..)`，否则调用`oopsBar(..)`。`baz(..)`也是一样。

本质上来说，没有哪一个是更恰当的。很多情况下，一个要比另一个更好。

无论哪一种情形，`foo(..)`返回的promise `p`都是用来控制接下来干什么。

此外，这两段代码在最后都对同一个promise `p`调用了两次 `then(..)`，是为了提前说明一点，即Promise（一旦解析）会永远保持同样的解析结果（fulfillment或rejection）不变，可以随后按需任意监听多次。

无论何时，一旦`p`解析了，下一步总是一样的，不论*现在*还是*以后*。

## Thenable 鸭子类型（Thenable Duck Typing）

在Promise领域中，一个重要的细节是如何判断某个值是否是一个真正的Promise。或者更直接一点，它是一个和Promise行为类似的值吗？

鉴于Promise是通过`new Promise(..)`语法构建的，你可能认为`p instanceof Promise`是一个可以接受的检查。但不幸的是，有一万种理由来证明这完全不够。

主要地，你可能从另一个浏览器窗口（iframe等）接收一个Promise，它可能有一个自己的Promise，不同于当前窗口/frame，此时，采用类型检查（即`p instanceof Promise`）无法识别Promise实例。

此外，库或者框架可能选择提供自己的Promise，而不是采用原生的ES6 Promise实现。事实上，你可能一直在没有Promise的旧浏览器上通过库使用Promise。

当我们在本章后面讨论Promise解析的时候，一个non-genuine-but-Promise-like（不是真正的但是像Promise）的值仍然能够识别和同化很重要，其原因会变得更明显。但目前，只记住我的话，这是这个谜团中很重要的一部分。

同样的，识别一个Promise（或者一个表现得像Promise）的方法是定义一个称为"thenable"的东西，它是指任何一个有`then(..)`方法的对象或者函数。假定任何此类的值是一个Promise-conforming thenable（符合Promise的thenable）。

假定一个值的“类型”是基于其形状（存在什么属性），通常这种“类型检查”称为“鸭子类型”--“如果看起来像鸭子，叫起来也像鸭子，那一定就是鸭子”。因此thenable的鸭子类型检查大概就是这样：

```javascript
if (
    p !== null &&
    (
        typeof p === "object" ||
        typeof p === "function"
    ) &&
    typeof p.then === "function"
) {
    // assume it's a thenable!
}
else {
    // not a thenable
}
```

Yuck(一脸嫌弃的眼神)!撇开这种丑陋的实现逻辑不谈（很多地方都采用这种方法），有些更深层次和更麻烦的东西还在后头。

如果你想用任一个恰巧有`then(..)`函数的对象/函数来fulfill一个Promise（译者注：指将Promise置为成功状态），但你并不想将它当做一个Promise/thenable，很不走运，因为它会被自动识别为thenable并且采用一些特殊的规则（见本章后面）。

如果是在你没意识到它有一个`then(..)`方法时，更是如此。比如：

```javascript
var o = { then: function(){} };

// make `v` be `[[Prototype]]`-linked to `o`
var v = Object.create( o );

v.someStuff = "cool";
v.otherStuff = "not so cool";

v.hasOwnProperty( "then" );     // false
```

`v`看起来一点也不像一个Promise或者thenable。它只是个有些属性的普通对象。你可能只是想将它和其它普通对象一样传递出去。

但你不知道的是，`v`也是`[[Prototype]]`链于另一个对象`o`，而`o`恰巧有一个`then(..)`方法。因此，thenable鸭子类型检查会认为`v`是一个thenable。呃。

甚至不需要像下面直接这样做：

```javascript
Object.prototype.then = function(){};
Array.prototype.then = function(){};

var v1 = { hello: "world" };
var v2 = [ "Hello", "World" ];
```

`v1`和`v2`都会被假定为thenable。你无法控制或者预测哪些代码有意或者无意地在`Object.prototype`，`Array.prototype`或者其它原生原型上添加了`then(..)`方法。如果指定的函数并没有调用任一个参数（译者注：指`then(..)`方法中的两个参数，一个是resolve，一个是reject）作为回调。那么任一个解析该值的Promise将会永远静默挂起！疯狂吧。

听起来不可思议或者不可能？也许吧。

但是请记住，在ES6之前，有几个知名的非Promise的库已经存在于社区了，并且恰巧其中有`then(..)`方法。其中一些库选择重新命名他们的方法来避免冲突（糟透了！）。其他的只是简单地降级到“与基于Promise的编程不兼容”的状态来应对他们无力改变使其不受影响这一局面。

标准决定劫持之前非保留的--并且听起来完全通用的--`then`属性名，这意味着，任何值（或者其代理）在过去、现在以及将来，都不能有`then(..)`方法出现，不论是有意的还是无意的，否则在Promise系统中，该值会被误认为thenable，进而很可能引起bug，不易追踪。

**警告：** 在Promise识别中，我不想以thenable的鸭子类型检查结束。还可以有其它选项，比如“品牌化（branding）”或者“反品牌化（anti-branding）”；似乎我们选择了最坏的一种折衷。但情况并非一团糟。thenable鸭子类型也是有用的，之后我们会看到。记住，如果把不是Promise的东西错误地识别为Promise，那样鸭子类型检查就会很危险。

## Promise信任问题（Promise Trust）

我们已经举了两个有力的例子（译者注：即作为未来值和完成事件），用来解释Promise能为我们的异步代码做些什么的不同方面。但如果我们止步于此，我们会错过Promise模式建立起来的一个最重要的特性：信任。

尽管我们已经在代码中很清楚地展开探究了*未来值*和*完成事件*，我们还不是很清楚Promise是如何设计的，能够解决我们在第二章“信任问题”一节中提出的所有*控制权反转*信任问题。只要稍加深究，我们就能在异步编程时重拾第二章中失去的信心！

让我们回顾一下只采用回调编程的信任问题。当传递一个回调给实体函数`foo(..)`，可能：

+ 太早调用回调
+ 太晚调用回调（或者从不调用）
+ 调用太多或者太少
+ 无法传递任何必要的环境/参数
+ 掩盖可能发生的任何错误/异常

Promise的诸多特性就是针对这些问题提供了有用的，经得起考验的答案。

### 太早调用（Calling Too Early）

这个问题主要在于代码是否会引起像Zalgo效应（见第二章），即一些任务有时是同步完成的，有时是异步完成的，这样会导致竞态。

Promise从定义上来说就不会受此影响。因为即使是一个立即的fulfilled Promise（比如`new Promise(function(resolve){ resolve(42); })`）就无法同步*监听*到。

也就是说，当你在一个Promise上调用`then(..)`方法时，即使Promise已经解析了，你提供给`then(..)`的回调函数总是**异步**执行的（要想知道更多，参考第一章“作业”一节）。

再也不需要插入`setTimeout(..,0)`的hack了，Promise自动阻止了Zalgo。

### 太晚调用（Calling Too Late）

与前一点类似，当Promise创建时调用`resolve(..)`或者`reject(..)`，Promise `then(..)`注册的监听回调是自动调度的。这些调度的回调函数会在下一个异步时刻触发（详见第一章中的“作业”）。

同步监听是不可能的，因此以这种方式运行的一串同步任务“推迟”（从效果上来说）另一个回调的执行是不可能的。也就是说，当一个Promise解析完，所有注册在`then(..)`上的回调函数都会在下一个异步窗口（再次，见第一章的“作业”）按序立即（immediately）调用，并且任一个回调内部发生的东西不会影响/推迟其它回调的执行。

例如：

```javascript
p.then( function(){
    p.then( function(){
        console.log( "C" );
    } );
    console.log( "A" );
} );
p.then( function(){
    console.log( "B" );
} );
// A B C
```

根据Promise定义的运行原理，此处`"C"`无法中断并领先`"B"`。

### Promise调度的怪癖（Promise Scheduling Quirks）

然而，有一点必须要注意，两个不同Promise的链式回调的相对顺序无法可靠预测时的调度有许多细微的差别。

如果两个promise `p1`和`p2`已经解析完了，那么`p1.then(..); p2.then(..)`最终`p1`的回调会在`p2`的回调之前被调用。但是有一些不可思议的情况，比如下面这个：

```javascript
var p3 = new Promise( function(resolve,reject){
    resolve( "B" );
} );

var p1 = new Promise( function(resolve,reject){
    resolve( p3 );
} );

var p2 = new Promise( function(resolve,reject){
    resolve( "A" );
} );

p1.then( function(v){
    console.log( v );
} );

p2.then( function(v){
    console.log( v );
} );

// A B  <-- not  B A  as you might expect
```

对此，我们之后会细讲，但如你所见，`p1`并不是立即值解析的，而是另一个promise `p3`，`p3`解析的是值`B`。这一特定行为是把`p3`*打开*传入到`p1`中，但是异步的，因此在异步作业队列（见第一章）中，`p1`的回调是在`p2`回调后面的。

为了避免这种微妙的噩梦，你绝不要依赖跨Promise来实现回调的排序/调度。实际上，一个好的方法是在多个回调的顺序很重要的时候，不要像这样编写代码。尽可能避免。

### 从不调用回调（Never Calling the Callback）

这是很常见的一个问题。可以用Promise采用多种方式解决。

首先。没有什么（甚至是一个JS错误）能够阻止Promise通知你它的解析结果（如果已经解析完了）。如果你为Promise同时注册了fulfillment和rejection回调，待Promise解析完成后，总有一个会被调用。

当然，如果回调函数本身有JS错误，你可能看不到期望的结果，但回调函数已经执行过了。我们接下来会讲在回调中如何接收错误，因为即使是那些也不会被掩盖。

但要是Promise本身永不解析呢？即使是那种情况，Promise也提供了解决方法，用一种更高级的抽象，称为“竞争（race）”：

```javascript
// a utility for timing out a Promise
function timeoutPromise(delay) {
    return new Promise( function(resolve,reject){
        setTimeout( function(){
            reject( "Timeout!" );
        }, delay );
    } );
}

// setup a timeout for `foo()`
Promise.race( [
    foo(),                  // attempt `foo()`
    timeoutPromise( 3000 )  // give it 3 seconds
] )
.then(
    function(){
        // `foo(..)` fulfilled in time!
    },
    function(err){
        // either `foo()` rejected, or it just
        // didn't finish in time, so inspect
        // `err` to know which
    }
);
```

这种Promise超时模式还有更多细节值得探究，之后会讨论。

重要的是，我们可以确保有一个信号作为`foo()`的结果，防止我们的程序被无限地挂起。

### 调用太少或太多次

从定义上看，就是回调被调用的次数。“太少”就是零调用，和我们之前说过的“从不”一个意思。

“太多”很容易解释，Promise被定义为只能解析一次。如果由于某些原因，Promise的创建代码试图多次调用`resolve(..)`或者`reject(..)`，或者两个都调用，Promise只会接受第一个解析，并静默地忽略之后的尝试。

因为Promise只能接卸一次，任一个`then(..)`注册的回调也只能调用一次（对每个而言）。

当然，如果你不止一次地注册同一个回调，（例如，`p.then(f); p.then(f);`），则注册多少次就执行多少次。响应函数只能调用一次并不能阻止你搬起石头砸自己的脚。

### 无法传递任何参数/环境（Failing to Pass Along Any Parameters/Environment）

Promise最多只有一个解析值（fulfillment或者rejection）。

如果你不显式地解析一个值，就会像传统JS一样，为`undefined`。但是一旦有值，则总会传到所有注册（fulfillment或者rejection）的回调中去，不论是*现在*的还是将来的。

记住：如果调用`resolve(..)`或者`reject(..)`的参数有多个时，除了第一个参数，所有后面的参数都会被默认忽略。看起来似乎有点违背我们之前描述的保证，其实并不是，因为这是对Promise机制的一个无效应用。其它一些无效的API应用（比如多次调用`resolve(..)`）也是同样受到*保护*的，因此Promise的行为是一致的。

如果你想传入多个值，必须把它们包在另一个你传入的单值里，比如一个`array`或者一个`object`。

至于环境，JS函数中总是保存着定义时的作用域闭包，因此，它们当然
可以访问任何你提供的环境状态。当然，只有回调的设计中也是这样的，因此这并不是Promise带来的额外好处--但能够保证我们可以依赖它。

### 掩盖任何错误/异常(Swallowing Any Errors/Exceptions)

从基本意义上来说，这是之前观点的重述。如果用一个*原因（reason）*（即错误信息）来将一个Promise置为失败状态，这个值（译者注：指reason）就会被传到rejection回调中去。

但此处还有更大的东西。在创建Promise过程中的任一时刻，或者在监听解析结果的时候，如果发生了JS异常错误，比如`TypeError`或者`ReferenceError`，异常会被捕获，并且会强制Promise变成失败状态。

比如：

```javascript
var p = new Promise( function(resolve,reject){
    foo.bar();  // `foo` is not defined, so error!
    resolve( 42 );  // never gets here :(
} );

p.then(
    function fulfilled(){
        // never gets here :(
    },
    function rejected(err){
        // `err` will be a `TypeError` exception object
        // from the `foo.bar()` line.
    }
);
```

发生自`foo.bar()`的的JS异常变成了一个Promise rejection，你可以捕获它并进行响应。

这是个重要的细节，因为能够有效地解决另一个潜在的Zalgo情形，即错误可能创建一个同步的响应而非错误则是异步的。Promise甚至把JS异常也转为异步的了，因此能够及大地减少竞态的发生。

但如果Promise被成功状态了，但是在监听过程中（在一个`then(..)注册回调中`）发生了JS异常，会发生什么呢？







