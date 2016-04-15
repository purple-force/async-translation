# 你不知道JS：异步

# 第三章：Promises

在第二章，我们指出了采用回调来表达异步和管理并发时的两种主要不足：缺乏序列化和可信度。既然我们对这些问题更熟悉了，是时候将注意力集中到解决这些问题的模式上了。

我们想要处理的第一个问题是*控制权反转*，信任很难维持，脆弱且容易丢失。

回想一下我们把程序的*延续*包裹在回调函数中，把回调交给其它方（甚至可能是外部代码），然后祈祷能够正确地调用回调函数。

我们这样做的原因是我们想说，“这个是在当前步骤结束之后，*后来*发生的”。

但要是我们能够反逆转*控制权反转*呢？要是能够让第三方返回一个结果，让我们知道它的任务结束时间，然后我们的代码决定接下来该做什么呢？而不是把程序的延续交给第三方。

这一范例称为**Promises**。

随着诸如开发人员和规范制定人员之类的人不顾一切地寻求解决代码/设计中回调地狱错乱问题的方法，Promises将会撼动整个JS世界。事实上，JS/DOM中新加入的API大多数都是建立在Promises上的。那么深入学习Promises可能是个不错的注意，不是吗？

**注意：** 本章中”立刻（immediately）“一词将会反复提及，通常指某些Promise的解析行为。然而，从本质上来说，”立刻“一词是根据作业队列（Job queue）（见第一章）来说的，不是*现在（now）*层面上严格同步的意思。

## Promise是什么？（What Is a Promise?）

当开发人员学习一种新的技术或者模式时，第一步通常是”给我看代码！“对我们而言，直接进入（译者注：指直接看代码）并学习很自然。

但结果是，只关注API的话，就会丢失一些抽象概念。Promise是这样的一种工具，如果不理解它是做什么的，只是学会使用API的话，使用起来会很痛苦。

因此，在我展示Promise代码之前，我想从概念上完完整整地解释Promise到底是什么。我希望这能够指导你更好地将Promise理论整合到自己的异步流中。

明确这点，让我们看一下两个关于Promise是什么的不同类比。

### 未来值（Future Value）

假设这个场景：我走到快餐店的柜台，点了一个奶酪汉堡，我递给收银员$1.47的零钱。通过下单和支付，我已经作了一次请求，希望有*值*（奶酪汉堡）返回。我已经开始了一次交易。

但通常，奶酪汉堡并不是立刻就能给我。收银员给了我一个代替奶酪汉堡的东西：一个有订单号的收据。这个订单号是一个IOU（”i owe you
“）的*承诺*，确保最后我能拿到我的奶酪汉堡。

我拿着我的收据和订单号。我知道它代表着我*未来的奶酪汉堡*，因此我再也不用担心了--除了我很饿之外！

当我在等的时候，我还可以做其它事，比如给我的朋友发短信说，”嗨，你能和我一起吃午餐吗？我要吃个奶酪汉堡。“

即使还没拿到手，我已经开始推演我的*未来奶酪汉堡*。我的大脑之所以能这样做是因为它把订单号当成了奶酪汉堡占位符。本质上这个占位符使得值*与时间无关*。它是**未来值**。

最终，我听到了，”113号！“我很高兴地拿着订单走向柜台。把订单递给收银员后，我拿到了我的奶酪汉堡。

换句话说，一旦我的*未来值*准备好了，我就可以拿承诺值（value-promise）换真正的值（value）了。

但还可能有另一种结果。他们叫了我的订单号，但是当我去取奶酪汉堡时，收银员很遗憾地告诉我，”很抱歉，我们恰巧卖完了奶酪汉堡“。抛开客户沮丧的场景来看，我们可以看到*未来值*的一个重要特点：它们既可能预示着成功，也可能预示着失败。

每次我点一个奶酪汉堡，我都知道可能最终我会得到一个，或者很遗憾地听到奶酪汉堡卖完了，不得不另寻其它作为午餐了。

**注意：** 在代码中，事情并不是如此简单，因为比方来说，订单号可能永远都不会被叫到，此时我们就无限地处在了悬而未决的状态。我们稍后处理这种情形。

#### 现在值和以后值（Values Now and Later）

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

不可否认，尽管有点难看，但是这种异步模式中还是有值得注意的地方。

在那段代码中，我们把`x`和`y`视作未来值，并且表示操作的`add(..)`并不关心`x`和`y`是否都立刻准备好了。换句话说，它统一了*现在*和*以后*，如此我们就可以依赖`add(..)`操作生成的一个可预测的结果。

通过使用时序一致的`add(..)`---*现在*和*以后*的行为都保持一致--使得异步代码更容易推演了。

更通俗一点来说：为了统一处理*现在*和*以后*，我们把它们都变成了*以后*。所有的操作都变成异步的了。

当然，这种简单的基于回调不是太令人满意。推演未来值而不需要担心时间维度，即值是否可用，对于这一好处而言，这只是一小步。

#### 承诺值（Promise Value）

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

**注意：** 在`add(..)`内部，`Promise.all([ .. ])`创建了一个promise（等待`promiseX`和`promiseY`解析）。`.then(..)`的链式调用创建了一个promise，`return values[0] + values[1]`立即得到解析（相加的结果）。因此，`add(..)`调用之后的链式`then(..)`调用--代码末尾--实际上是对返回的第二个promise进行操作，而不是对`Promise.all([ .. ])`创建的第一个。另外，尽管我们没有对第二个`then(..)`进行链式操作，它其实也生成了一个promise。这种Promise的链式调用会在本章后面作更详尽的说明。

就像奶酪汉堡订单一样，一个Promise的解析结果可能不是成功（fulfillment）而是失败（rejection）的。不像一个成功的promise，其值总是程序化的（programmatic，译者注：指通过程序设定），一个失败值--通常称为“失败原因”--既可通过程序逻辑直接设定，又可由隐式的运行异常导致。

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

因为Promise从外面封装了跟时间无关的状态--等待潜在值的成功或失败--Promise本身是跟时间无关的，因此Promise可以以可预测的方式组织（组合）起来，不管时间或结果。

此外，一旦Promise解析了，其状态永远不变--自那一时刻起，变成了*不可更改的值*--可以按需多次监听。

**注意：** 因为Promise一旦解析，就无法从外面更改其状态了，这样把值传递给第三方就很安全，因为无法无意或恶意地修改它。尤其是当有多个当事方监听这一个Promise的解析结果时更是如此。一方无法影响另一方对Promise解析的监听。不可变性听起来似乎像一个学术课题，但它其实是Promise设计中十分重要的方面之一，不能随意忽略。

那是理解Promise所需的最强大和最重要的概念之一。有一定实践经验后，你也能通过难看的回调组合来专门实现同样的效果，但那并不是一个真正有效地策略，尤其是你必须一次又一次地这么做。

Promise是一种很容易封装和构成*未来值*的可重复性机制。

### 完成事件（Completion Event）

正如我们刚刚看到的，一个Promise的行为就像一个*未来值*。但还可以采用另一种方式来思考Promise：在异步任务的两步或多步中--一个时序的this-then-that--作为一种流控制机制。

假设调用函数`foo(..)`来执行某个任务。我们不知道任何细节，也不关心。可能是立即完成的，也可能花一些时间。

我们只需简单地知道`foo(..)`什么时候完成，以便我们能够转到下一个任务中去。换句话说，我们想要通过某种方式来通知我们`foo(..)`已经完成了，以便于*继续*做其它事。

在传统JS中，如果你需要监听某个通知（notification），你可能会想到事件。因此，我们可以重构通知需求为需要监听`foo(..)`发出的*完成*（或者*继续*）事件。

**注意：** 称之为“完成事件”还是“继续事件”取决于你自己。关注点更应该放在`foo(..)`发生了什么或者`foo(..)`后会发生什么。这两个角度都是准确和有用的。事件通知告诉我们`foo(..)`已经完成了，同样也可以说是能够进行下一步了。事件通知时调用的传入的回调函数本身就是我们之前提到的*延续(continuation)*。因为*完成事件*更关注于`foo(..)`,此时我们也更关注于它。在本篇其余部分，会倾向于采用*完成事件*。

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

`foo(..)`显式地创建了事件订阅功能用作返回，然后调用代码接收它并
针对它注册了两个事件处理函数。

很明显，这是从正常的回调导向的代码反转（控制权），也是特意这么做的。`foo(..)`返回一个称之为`evt`的事件功能，用它来接收回调函数。

但是如果你回想一下第二章，回调函数本身代表*控制权反转（inversion of control）*。因此，反转回调模式通常称之为*反转的反转（inversion of inversion）*，或者*不反转控制权（uninversion of control）*--在我们需要以它为主的地方恢复调用代码的控制权。

这样做的一大好处就是代码的多个独立部分可以获得事件监听能力，当`foo(..)`完成的时候，它们可以独立地接到通知并进行后续步骤：

```javascript
var evt = foo( 42 );

// let `bar(..)` listen to `foo(..)`'s completion
bar( evt );

// also, let `baz(..)` listen to `foo(..)`'s completion
baz( evt );
```

*不反转控制权（Uninversion of control）*能够实现更好的关注点分离，`bar(..)`和`baz(..)`不需要关心`foo(..)`是如何调用的。同样的，`foo(..)`也不需要知道或者关心`bar(..)`和`baz(..)`的存在或者`foo(..)`完成时`bar(..)`和`baz(..)`正等着被通知。

本质上来说，这个`evt`对象是不同关注点之间的一个中立的第三方协商。

#### Promise “事件”（Promise "Events"）

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

当我们在本章后面讨论Promise解析过程的时候，一个non-genuine-but-Promise-like（不是真正的但是像Promise）的值仍然能够识别和理解是很重要的，其原因会变得更明显。但目前，只记住我的话，这是谜团中很重要的一部分。

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

Yuck(一脸嫌弃的眼神)!撇开这种难看的实现逻辑不谈（很多地方都采用这种方法），有些更深层次和更麻烦的东西还在后头。

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

我们已经举了两个有力的例子（译者注：即作为未来值和完成事件），用来从不同方面解释Promise能为我们的异步代码做些什么。但如果我们止步于此，我们会错过Promise模式建立起来的一个最重要的特性：信任。

尽管我们已经在代码中很清楚地展开探究了*未来值*和*完成事件*，我们还不是很清楚Promise是如何设计的，能够解决我们在第二章“信任问题”一节中提出的所有*控制权反转*信任问题。只要稍加深究，我们就能在异步编程时重拾第二章中失去的信心！

让我们回顾一下只采用回调编程的信任问题。当传递一个回调给utility`foo(..)`，可能：

+ 太早调用回调
+ 太晚调用回调（或者从不调用）
+ 调用太多或者太少
+ 无法传递任何必要的环境/参数
+ 掩盖可能发生的任何错误/异常

Promise的诸多特性就是针对这些问题提供了有用的，经得起考验的答案。

### 太早调用（Calling Too Early）

这个问题主要在于代码是否会引起像Zalgo效应（见第二章），即一些任务有时是同步完成的，有时是异步完成的，这样会导致竞态。

Promise从定义上来说就不会受此影响。因为即使是一个立即的fulfilled Promise（比如`new Promise(function(resolve){ resolve(42); })`）也无法同步*监听*到。

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

#### Promise调度的怪癖（Promise Scheduling Quirks）

然而，有一点必须要注意，在相对顺序无法可靠预测时，两个不同Promise的链式回调的调度有许多细节需要考虑。

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

### 调用太少或太多次（Calling Too Few or Too Many Times）

从定义上看，就是回调被调用的次数。“太少”就是零调用，和我们之前说过的“从不”一个意思。

“太多”很容易解释，Promise被定义为只能解析一次。如果由于某些原因，Promise的创建代码试图多次调用`resolve(..)`或者`reject(..)`，或者两个都调用，Promise只会接受第一个解析，并静默地忽略之后的尝试。

因为Promise只能解析一次，任一个`then(..)`注册的回调也只能调用一次（对每个而言）。

当然，如果你不止一次地注册同一个回调，（例如，`p.then(f); p.then(f);`），则注册多少次就执行多少次。响应函数只能调用一次并不能阻止你搬起石头砸自己的脚。

### 无法传递任何参数/环境（Failing to Pass Along Any Parameters/Environment）

Promise最多只有一个解析值（fulfillment或者rejection）。

如果你不显式地解析一个值，就会像传统JS一样，为`undefined`。但是一旦有值，则总会传到所有注册（fulfillment或者rejection）的回调中去，不论是*现在*的还是将来的。

记住：如果调用`resolve(..)`或者`reject(..)`时的参数有多个，除了第一个参数，所有后面的参数都会被默认忽略。看起来似乎有点违背我们之前描述的保证，其实并不是，因为这是对Promise机制的一个无效应用。其它一些无效的API应用（比如多次调用`resolve(..)`）也是同样受到*保护*的，因此Promise的行为是一致的。

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

发生自`foo.bar()`的JS异常变成了一个Promise rejection，你可以捕获它并进行响应。

这是个重要的细节，因为能够有效地解决另一个潜在的Zalgo情形，即错误可能创建一个同步的响应而非错误（nonerrors）则是异步的。Promise甚至把JS异常也转为异步的了，因此能够极大地减少竞态的发生。

但如果Promise被置为成功状态（fulfilled）了，但是在监听过程中（在一个`then(..)`注册回调中）发生了JS异常，会发生什么呢？即使能捕获这些异常，在更深入了解之前，你可能会对处理异常的方式感到惊讶。

```javascript
var p = new Promise( function(resolve,reject){
    resolve( 42 );
} );

p.then(
    function fulfilled(msg){
        foo.bar();
        console.log( msg ); // never gets here :(
    },
    function rejected(err){
        // never gets here either :(
    }
);
```

等等，似乎`foo.bar()`的异常被掩盖了。别担心，并没有。但是更深层次的东西出问题了，即我们无法监听这个异常了。`p.then(..)`本身返回了另一个promise，这个promise会因`TypeError`异常而被置为失败状态（rejected）。

为什么没有直接调用已经定义好的错误处理函数呢？表面上看起来似乎很合逻辑。但会违背一条重要原则，即一旦解析，Promise就是**不可改变的**。`p`已经由值`42`置为成功状态，因此之后不可能仅仅因为监听`p`的解析中有一个异常而改为失败状态。

除了违背这一原则外，这样的行为也可能造成很大破坏，假设在promise `p`上有许多`then(..)`注册回调，因为有些会调用，而有些不会，这样会导致原因不清不楚。

### 可信赖的Promise？（Trustable Promise?）

基于Promise模式建立信任还有最后一个细节。

毫无疑问，你已经注意到Promise并没有摆脱回调。它们只是改变了回调传入的地方。我们从`foo(..)`中得到一个东西（看起来是一个真正的Promise），然后把回调传入其中，而不是把回调传入到`foo(..)`中。

但为什么这比只使用回调更值得信赖呢？我们如何确保得到的东西真的是可信赖的Promise呢？我们所相信的（仅仅因为我们已经相信它了）是不是基本上都是空中楼阁？

关于Promise，一个很重要但经常被忽略的细节是，Promise对这个问题也有一个解决方案。即包含在原生ES6 `Promise`中的`Promise.resolve(..)`。

如果你向`Promise.resolve(..)`中传入一个立即值、非Promise值、非thenable值，你会得到一个以该值将状态置为成功的promise。换句话说，以下两个promise `p1`和`p2`行为基本上一致：

```javascript
var p1 = new Promise( function(resolve,reject){
    resolve( 42 );
} );

var p2 = Promise.resolve( 42 );
```

但是如果你向`Promise.resolve(..)`中传入一个真正的Promise，你只会得到同一个promise：

```javascript
var p1 = Promise.resolve( 42 );

var p2 = Promise.resolve( p1 );

p1 === p2; // true
```

更重要的是，如果你向`Promise.resolve(..)`中传入一个非Promise的thenable值。它会试图拆析（unwrap）这个值，并且会一直持续直至抽取到一个具体的non-Promise-like值。

回想一下我们之前讨论的thenable？

如下：

```javascript
var p = {
    then: function(cb) {
        cb( 42 );
    }
};

// this works OK, but only by good fortune
p
.then(
    function fulfilled(val){
        console.log( val ); // 42
    },
    function rejected(err){
        // never gets here
    }
);
```

这个`p`是个thenable。但不是一个真正的Promise。幸运的是，这是合理的，因为绝大多数都是这样的。但要是你得到的是这样的呢：

```javascript
var p = {
    then: function(cb,errcb) {
        cb( 42 );
        errcb( "evil laugh" );
    }
};

p
.then(
    function fulfilled(val){
        console.log( val ); // 42
    },
    function rejected(err){
        // oops, shouldn't have run(本不该运行的啊！)
        console.log( err ); // evil laugh
    }
);
```

这个`p`是一个thenable，但并没有如promise表现得那么好。它是恶意的吗？或者仅仅是忘记了Promise是如何运行的？说实话，没关系。无论哪一种情形（即以上两个例子）都是不可信任的。

然而，我们可以将这两个版本的`p`传入到`Promise.resolve(..)`中，最终会得到我们期望的标准、安全的结果。

```javascript
Promise.resolve( p )
.then(
    function fulfilled(val){
        console.log( val ); // 42
    },
    function rejected(err){
        // never gets here
    }
);
```

`Promise.resolve(..)`会接收任何thenable，然后将其拆析（unwrap）直至获得一个非thenable值。但是从`Promise.resolve(..)`，你会得到一个真正的promise，**一个你可以信赖的promise**。如果你传入的已经是个真正的promise，只会原样返回，因此，通过`Promise.resolve(..)`过滤来获取信任一点坏处也没有。

因此，假设我们正在调用`foo(..)`utility，我们不确定它的返回值是否是正常的Promise，但我们知道它至少是个thenable。`Promise.resolve(..)`会给我们一个值得信赖的Promise包装用作链式调用：

```javascript
// don't just do this:
foo( 42 )
.then( function(v){
    console.log( v );
} );

// instead, do this:
Promise.resolve( foo( 42 ) )
.then( function(v){
    console.log( v );
} );
```

**注意：** 通过`Promise.resolve(..)`包装任何函数返回值（thenable或者其它）的另一个好处是，很容易将一个函数调用标准化为一个表现良好的异步任务。如果`foo(42)`有时返回一个立即值，有时返回一个Promise，`Promise.resolve( foo(42) )`能够确保它总是一个Promise返回值。避免Zalgo让代码更好。

### 信任建立（Trust Built）

希望前面的讨论完全“resolve”（双关，既指解决，又指Promise中的resolve）你心中的疑惑，即为什么Promise是可信赖的，更重要的是，为什么在构造鲁棒、可维护的软件时，信任是那么重要。

在JS中，你能在没有信任的情况下编写异步代码吗？当然，你可以。我们JS开发者已经只用回调异步编程快二十年了。

你对你所建立之上的机制信任到什么程度，才能够使之可预测和可依赖，一旦你开始质疑，你就会渐渐意识到回调的信任根基并不十分牢固。

Promise是以可信赖语义增强回调的一种模式，因此其行为更合理，更值得信赖。通过反逆转回调的*控制权反转*，我们采用专门用来健全异步的可信赖系统（Promise）来进行控制。

## 链式流（Chain Flow）

我们已经暗示过好多次了，Promise不仅仅是一个单步的*this-then-that*操作。当然，那是一个构建块，但是结果表明，我们可以串起多个Promise来代表一系列的异步步骤。

成功的关键在于Promise的两个内在的行为：

+ 每次对Promise调用`then(..)`时，都会创建并返回一个新的Promise，我们可以对其进行链式操作。
+ `then(..)`调用的fulfillment回调（第一个参数）返回的任何值都会自动设置为*链式*Promise（见第一点）的fulfillment。

让我们首先说明一下是什么意思，之后我们会明白Promise是如何帮助我们创建异步序列控制流的。考虑如下代码：

```javascript
var p = Promise.resolve( 21 );

var p2 = p.then( function(v){
    console.log( v );   // 21

    // fulfill `p2` with value `42`
    return v * 2;
} );

// chain off `p2`
p2.then( function(v){
    console.log( v );   // 42
} );
```

通过返回`v * 2`(即`42`),我们将第一个`then(..)`调用生成的promise `p2`置成成功状态，当调用`p2`的`then(..)`时，它从`return v * 2 `语句接收fulfillment。当然，`p2.then(..)`创建了另一个promise，我们可将它存储在`p3`变量中。

但是必须创建中间变量`p2`(或者`p3`等)有点恼人。值得庆幸的是，我们可以简单地将它们串联起来：

```javascript
var p = Promise.resolve( 21 );

p
.then( function(v){
    console.log( v );   // 21

    // fulfill the chained promise with value `42`
    return v * 2;
} )
// here's the chained promise
.then( function(v){
    console.log( v );   // 42
} );
```

那么现在第一个`then(..)`就是异步序列中的第一步，第二个`then(..)`就是第二步。只要我们需要，就可以一直往下扩展。只要通过自动生成的Promise，链接到前一个`then(..)`上就行了。

但此处似乎少了什么东西。要是我们想让步骤2等待步骤1作一些异步操作呢？我们采用的是立即的（immediately）`return`语句，会立刻将链式的promise置为成功状态。

回想一下，当你传给它的是一个Promise或者thenable而不是一个最终值时（译者注：此处指立即值），`Promise.resolve(..)`是如何运行的，这是使得一个Promise序列在每一步都具有异步能力的关键。`Promise.resolve(..)`会直接返回接收的真正Promise，或者拆析（unwrap）接收的thenable的值--并且在拆析（unwrap）thenable时会一直递归下去。

如果你从fulfillment（或者rejection）的回调函数中`return`一个thenable或者Promise时，也会发生同样的拆解。

考虑如下：

```javascript
var p = Promise.resolve( 21 );

p.then( function(v){
    console.log( v );   // 21

    // create a promise and return it
    return new Promise( function(resolve,reject){
        // fulfill with value `42`
        resolve( v * 2 );
    } );
} )
.then( function(v){
    console.log( v );   // 42
} );
```

即使我们将`42`包装到了返回的promise中，它仍然会被拆析（unwrap）并最终作为链式promise的解析项，以便第二个`then(..)`仍然接收到`42`。如果我们将异步引入到那个包装promise中，一切照旧：

```javascript
var p = Promise.resolve( 21 );

p.then( function(v){
    console.log( v );   // 21

    // create a promise to return
    return new Promise( function(resolve,reject){
        // introduce asynchrony!
        setTimeout( function(){
            // fulfill with value `42`
            resolve( v * 2 );
        }, 100 );
    } );
} )
.then( function(v){
    // runs after the 100ms delay in the previous step
    console.log( v );   // 42
} );
```

好强大！现在我们可以按需构建任意多的异步步骤，并且每一步都可以按需推迟（或不推迟）下一步的执行。

当然，这些例子中步骤之间传递的值是可选的。如果你不返回一个显式的值，则会假定有个隐式的`undefined`，并且promise还是以同样的方式串起来。每个Promise的解析项只是进行到下一步的信号。

为了更进一步说明链式，让我们创建一个通用utility，用来生成延时Promise，使之能够多步复用：

```javascript
function delay(time) {
    return new Promise( function(resolve,reject){
        setTimeout( resolve, time );
    } );
}

delay( 100 ) // step 1
.then( function STEP2(){
    console.log( "step 2 (after 100ms)" );
    return delay( 200 );
} )
.then( function STEP3(){
    console.log( "step 3 (after another 200ms)" );
} )
.then( function STEP4(){
    console.log( "step 4 (next Job)" );
    return delay( 50 );
} )
.then( function STEP5(){
    console.log( "step 5 (after another 50ms)" );
} )
...
```

调用`delay(200)`会创建一个200ms后fulfill的promise，然后从第一个`then(..)`的fulfillment回调中返回，这会让第二个`then(..)`的promise等那个200ms的promise。

**注意：** 如上所述，在交替过程中有两个promise：200ms延时的promise和来自第二个`then(..)`所链的promise（译者注：指第一个`then(..)`生成的promise）。但是你从心理上觉得将这两个promise整合起来会更容易理解，Promise机制自动为你整合状态。从那个角度讲，你可以认为`return delay(200)`创建了一个promise并替代了早先返回的链式promise。

然而，老实说，没有信息传递的序列延时并不是一个特别有用的Promise流控制的例子。让我们看一个更有实际意义的场景。

让我们考虑Ajax请求，而不是定时器：

```javascript
// assume an `ajax( {url}, {callback} )` utility

// Promise-aware ajax
function request(url) {
    return new Promise( function(resolve,reject){
        // the `ajax(..)` callback should be our
        // promise's `resolve(..)` function
        ajax( url, resolve );
    } );
}
```

我们首先定义了一个`request(..)`utility，用来构建一个promise代表`ajax(..)`调用的完成：

```javascript
request( "http://some.url.1/" )
.then( function(response1){
    return request( "http://some.url.2/?v=" + response1 );
} )
.then( function(response2){
    console.log( response2 );
} );
```

**注意：** 开发者常遇到的一种情形是，他们想采用一些本身不支持Promise（Promise-enabled）的第三方utility（比如此处的`ajax(..)`，它需要一个回调函数）来实现类Promise（Promise-aware）的异步流控制。尽管原生的ES6 `Promise`机制无法自动为我们提供这种模式，但是所有的Promise库会提供。通常称这个过程为“提升（lifting）”或者“promise化（promisifying）” 或者其它变体。我们之后会讨论这一技术。

我们通过第一个URL调用`request(..)`（能返回Promise（Promise-returning））来隐式创建链的第一步，然后用第一个`then(..)`链接到返回的promise上。

一旦`response1`返回，我们用那个值来构建第二个URL，作第二次`request(..)`调用。第二个`request(..)`返回promise后，异步流中的第三步等待Ajax调用完成。最终，一旦返回值，立即打印`response2`。

我们构建的Promise链不仅仅是一个表示多步异步序列的流控制，还充当了步与步之间传递信息的通道。

要是Promise链中的某一步出错了呢？错误/异常是基于单个Promise的（译者注：指异常发生在生成Promise的每个链中），这意味着可以在链中的任一点捕获这个错误，并且那个捕获在那时充当了“重设”链回到正常操作的角色:

```javascript
// step 1:
request( "http://some.url.1/" )

// step 2:
.then( function(response1){
    foo.bar(); // undefined, error!

    // never gets here
    return request( "http://some.url.2/?v=" + response1 );
} )

// step 3:
.then(
    function fulfilled(response2){
        // never gets here
    },
    // rejection handler to catch the error
    function rejected(err){
        console.log( err ); // `TypeError` from `foo.bar()` error
        return 42;
    }
)

// step 4:
.then( function(msg){
    console.log( msg );     // 42
} );
```

当step2中发生错误时，step3中的rejection回调捕获到该异常。如果rejection函数中有返回值（此处代码中为`42`），则为step4将promise置为成功状态，以便让链回到fulfillment状态。

**注意：** 正如我们早前讨论的一样，当从一个fulfillment函数中返回一个promise时，它会被拆析（unwrap）并且推迟下一步。这同样也适用于rejection函数，即如果step3返回一个promise而不是`return 42`，则那个promise会推迟step4。一个`then(..)`中的fulfillment和rejection回调中的异常都会将下一个（链式）promise立即置为失败状态。

如果你在一个promise上调用`then(..)`，并且你只传了一个fulfillment回调，则会替换为一个假定的rejection处理函数：

```javascript
var p = new Promise( function(resolve,reject){
    reject( "Oops" );
} );

var p2 = p.then(
    function fulfilled(){
        // never gets here
    }
    // assumed rejection handler, if omitted or
    // any other non-function value passed
    // function(err) {
    //     throw err;
    // }
);
```

如你所见，假定的rejection处理函数只是简单地重新抛出错误，最终强制`p2`（链式的promise）以同样的错误原因reject。本质上，这允许错误沿着Promise链继续传播，直至遇到一个显式定义的rejection处理函数。

**注意：** 关于错误处理，稍后我们会讨论更多细节，因为有些其它微妙的细节需要关注。

如果没向`then(..)`中传递一个有效的fulfillment处理函数，则同样也会替换为一个默认的处理函数：

```javascript
var p = Promise.resolve( 42 );

p.then(
    // assumed fulfillment handler, if omitted or
    // any other non-function value passed
    // function(v) {
    //     return v;
    // }
    null,
    function rejected(err){
        // never gets here
    }
);
```

如你所见，默认的fulfillment处理函数只是简单地将接收到的值传递到下一步（Promise）。

**注意：** `then(null,function(err){ .. })`模式--只处理rejection（如果有的话）而让fulfillment通过--有一个快捷的API：`catch(function(err){ .. })`。我们将在下一节中详尽地讨论`catch(..)`。

让我们简单回顾下Promise支持的链式流控制的固有行为：

+ 对一个Promise调用`then(..)`会自动生成一个新的Promise并返回。
+ 在fulfillment/rejection处理函数内部，如果返回一个值或者抛出一个异常，则新返回的（链式）Promise会相应地得到解析。
+ 如果fulfillment或者rejection处理函数返回一个Promise，则该Promise会被拆析（unwrap），它的解析结果会成为当前`then(..)`返回的链式Promise的解析结果。

尽管链式流控制很有用，但最确切地说，这是Promise组成（组合）方式的一个附加好处，而不是主要目的。正如我们多次讨论过的，Promise将异步标准化并且封装了与时间无关的值状态，那才是我们得以用这种有用的方式将它们链在一起的原因。

当然，在处理第二章我们提出的回调混乱问题时，链式的序列化表示（this-then-this-then-this...）是个巨大改善。但是仍然有相当数量的样板（`then(..)`和`function(){ .. }`）需要费力的读完。在下一章，我们将会见到一个明显更好的、采用生成器的序列化流控制表示。

### 术语：解析，成功，失败（Terminology: Resolve, Fulfill, and Reject）

在更深入学习Promise前，我们需要搞清楚一些有点混淆的术语，“resolve”，“fulfill”和“reject”。首先考虑`Promise(..)`构造函数：

```javascript
var p = new Promise( function(X,Y){
    // X() for fulfillment
    // Y() for rejection
} );
```

如你所见，两个回调函数（此处标为`X`和`Y`）。第一个*通常*用于将Promise标为fulfilled状态，第二个*总是*将Promise标为rejected状态。但“通常”是指什么？准确地命名这些参数（译者注：指`X`和`Y`）意味着什么？

最终，只是你的用户代码对引擎有用，标识符名称对引擎而言没任何意义。因此从技术上来说，标识符名称并不重要，`foo(..)`和`bar(..)`同样可以。但是你所用的词不但影响你如何思考这段代码，而且也影响团队中的其他开发人员。对精心安排的异步代码的错误思考比意大利面条式的回调更糟。

因此，如何称呼它们真的有点重要。

第二个参数很容易确定。几乎所有的文章采用`reject(..)`作为它的名字，因为这就是它（也是唯一！）做的事，是个非常好的命名选择。我强烈建议你总是使用`reject(..)`。

但是关于第一个参数，就有点模糊不清了，在Promise文献中通常称为`resolve(..)`，那个词很明显和“resolution”相关，即在所有文献中（包括这本书）用来描述给一个Promise设置最终值/状态。我们已经好几次使用"resolve the Promise"来表示fulfill或者reject一个Promise。

但是，如果这个参数看起来似乎是专门用于fulfill Promise，为什么我们不叫它`fulfill(..)`，而叫它`resolve(..)`更准确呢？为了回答这个问题，我们也来看一下两个`Promise` API：

```javascript
var fulfilledPr = Promise.resolve( 42 );

var rejectedPr = Promise.reject( "Oops" );
```

`Promise.resolve(..)`创建了一个按所给值解析的Promise。在这个例子中，`42`是个正常的，非Promise，非thenable的值。因此，以值`42`创建了fulfilled的promise `fulfilledPr`。`Promise.reject("Oops")`以原因短语`"Oops"`创建了一个rejected的promise `rejectedPr`。

现在让我们举例说明一下，如果显式地用在可能导致fulfillment或者rejection的上下文中，为什么“resolve”一词（比如 `Promise.resolve(..)`中）很清晰并且确实更准确：

```javascript
var rejectedTh = {
    then: function(resolved,rejected) {
        rejected( "Oops" );
    }
};

var rejectedPr = Promise.resolve( rejectedTh );
```

正如本章早些时候讨论的那样，`Promise.resolve(..)`会直接返回接收的真实Promise，或者拆析（unwrap）接收的thenable。如果拆析（unwrap）的thenable显示的是rejected状态，则`Promise.resolve(..)`事实上返回的是同样的rejected状态。

因此，`Promise.resolve(..)`是个很好的，准确的API方法名称，因为它实际上既能生成fulfillment，又能生成rejection。

`Promise(..)`构造函数的第一个回调参数既会拆析（unwrap）一个thenable（与`Promise.resolve(..)`一致），也可拆析（unwrap）一个真正的Promise：

```javascript
var rejectedPr = new Promise( function(resolve,reject){
    // resolve this promise with a rejected promise
    resolve( Promise.reject( "Oops" ) );
} );

rejectedPr.then(
    function fulfilled(){
        // never gets here
    },
    function rejected(err){
        console.log( err ); // "Oops"
    }
);
```

现在应该明白`resolve(..)`是`Promise(..)`构造函数的第一个回调参数的合适名称了吧。

**警告：** 前面提到的`reject(..)`**并不**会像`resolve(..)`一样进行拆析。如果你向`reject(..)`中传入一个Promise/thenable。则该未处理的值将被设为rejection原因短语。随后的rejection处理函数会接收到你传入`reject(..)`的Promise/thenable，而不是最终拆析的立即值。

现在让我们的注意力回到`then(..)`中的回调。它们应该称为什么（无论是在文献中还是在代码中）？我建议`fulfilled(..)`和`rejected(..)`：

```javascript
function fulfilled(msg) {
    console.log( msg );
}

function rejected(err) {
    console.error( err );
}

p.then(
    fulfilled,
    rejected
);
```

`then(..)`中的第一个参数，毫无疑问总是fulfillment情况，因此没有 必要使用二义性术语“resolve”。顺便提一下，ES6规范中使用`onFulfilled(..)`和`onRejected(..)`来标记这两个回调，因此使用这两个术语很准确的。

## 错误处理（Error Handling）

在异步编程中，关于Promise rejection--可能是通过主动的`reject(..)`调用，也可能是通过偶然的JS异常--是如何实现健全的错误处理，我们已经看了几个例子了。

对绝大多数开发者而言，错误处理最自然的形式就是同步的`try..catch`结构了。不幸的是，它只是同步的，因此无法在异步代码模式中奏效：

```javascript
function foo() {
    setTimeout( function(){
        baz.bar();
    }, 100 );
}

try {
    foo();
    // later throws global error from `baz.bar()`
}
catch (err) {
    // never gets here
}
```

`try..catch`固然很好，但是在异步操作中不起作用。也就是说，除非有一些附加的环境支持，我们会在第四章的生成器中讨论。

在回调中，一些标准中已经出现了模式化错误处理，绝大多数是“错误优先回调（error-first callback）”类型的：

```javascript
function foo(cb) {
    setTimeout( function(){
        try {
            var x = baz.bar();
            cb( null, x ); // success!
        }
        catch (err) {
            cb( err );
        }
    }, 100 );
}

foo( function(err,val){
    if (err) {
        console.error( err ); // bummer :(
    }
    else {
        console.log( val );
    }
} );
```

**注意：** 此处的`try..catch`只在`baz.bar()`调用是同步的、立即成功或者失败时才起作用。如果`baz.bar()`本身就是异步完成函数，则其中的任何异步错误都无法捕获。

我们传给`foo(..)`的回调通过保留第一个参数`err`，希望接收一个错误信号。如果存在，则假定有错误发生。如果不存在，则假定成功。

此种错误处理在技术上称为*async capable*，但这一点都不好。多级错误优先回调和无处不在的`if`语句检查交织在一起，不可避免地将你置于回调地狱的危险之中（见第二章）。

让我们回到Promise的错误处理中来，采用传给`then(..)`的rejection处理函数的方式。Promise并没有采用流行的“错误优先回调”的设计方式，而是采用“分隔回调”的方式，一个是fulfillment回调，一个是rejection回调：

```javascript
var p = Promise.reject( "Oops" );

p.then(
    function fulfilled(){
        // never gets here
    },
    function rejected(err){
        console.log( err ); // "Oops"
    }
);
```

尽管这种模式的错误处理表面上看起来很好理解，但是Promise错误处理的细节通常更难完全掌握。

考虑如下代码：

```javascript
var p = Promise.resolve( 42 );

p.then(
    function fulfilled(msg){
        // numbers don't have string functions,
        // so will throw an error
        console.log( msg.toLowerCase() );
    },
    function rejected(err){
        // never gets here
    }
);
```

如果`msg.toLowerCase()`正常的抛出一个错误（确实会），为什么我们的错误处理函数没收到通知呢？正如早先解释的一样，因为那个错误处理是为`p` promise，已经由值`42`变为fulfilled状态了。`p` promise 是不可改变的，因此，唯一一个能被通知到错误的是`p.then(..)`返回的promise，而在该例中我们没有去捕获。

这向我们描述了一幅很清楚的画面，即为什么Promise的错误处理容易出错（双关语）。错误太容易被掩盖了，很少是出于你的意愿。

**警告：** 如果以非法的方式使用Promise API，并且错误阻止了正常的Promise构建，则会立即抛出异常，**而不是一个rejected Promise**。一些不正确的使用导致Promise构建失败的例子有：`new Promise(null)`，`Promise.all()`，`Promise.race(42)`等。如果不首先采用Promise API构建一个合法的Promise，你无法得到一个rejected Promise！

### 绝望的深渊（Pit of Despair）

Jeff Atwood 多年前提过：编程语言通常是这样设置的，即默认开发者会掉进”绝望的深渊“（[http://blog.codinghorror.com/falling-into-the-pit-of-success/](http://blog.codinghorror.com/falling-into-the-pit-of-success/)）--会遭受惩罚--并且你不得不花更大的力气去修正它。他恳求我们创建”成功之坑（pit of success）“，即默认你会成功，并且必须花大力气才会失败。

Promise 的错误处理毫无疑问是”Pit of Despair“式的设计。默认情况下，假定你想让Promise状态掩盖任何错误，并且如果你忘记监听那个状态，那么错误就会静默地消逝--通常令人绝望。

为了避免丢掉那个错误，已经有一些开发人员声称Promise链的”最佳实践“是总以一个`catch(..)`结尾，比如：

```javascript
var p = Promise.resolve( 42 );

p.then(
    function fulfilled(msg){
        // numbers don't have string functions,
        // so will throw an error
        console.log( msg.toLowerCase() );
    }
)
.catch( handleErrors );
```

因为没给`then(..)`传递rejection处理函数，所以就替换为默认的处理函数，即简单的将错误传播到链中的下一个promise。这样，在解析时，`p`中的错误和`p`之后的错误（如`msg.toLowerCase()`）会被过滤到最终的`handleErrors(..)`中。

问题解决了，是吗？没那么快！

如果`handleErrors(..)`本身也有错误，会发生什么呢？谁来捕获那个错误？仍然有一个未参与的promise：`catch(..)`返回的，我们没有捕获，也没有为其注册rejection处理函数。

你不能简单地在链尾再接个`catch(..)`，因为同样有可能失败。在任何Promise链的最后一步，无论是什么，总可能在未监听的promise中悬着一个未捕获的错误。

听起来似乎是个不可能的难题？

### 未捕获处理（Uncaught Handling）

这并不是个容易完全解决的问题。有些其它方法，许多人说可能更好一点。

一些Promise库已经添加了一些方法，用来注册类似于”全局的未处理rejection“处理函数的东西，它会被调用，而不是全局地抛出异常。但对于如何识别一个错误是未处理的，他们的解决方案是用一个随机长度的定时器，比如3秒，从rejection时开始运行。如果在定时器触发前，一个Promise被rejected了，但是没有注册错误处理函数，之后就会假定你不会给它注册处理函数，因此它是”未捕获的（uncaught）“。

在实际开发中，对许多库而言，这种做法很奏效。因为绝大多数使用模式的Promise rejection和监听rejection之间不需要太长的延时。但是这种模式有些问题，因为3秒太随意了（即使是从经验上来看），并且因为有些情况下，确实需要Promise在一定时间内保持rejected状态，你并不希望所有误报（false positives）（还没处理的”未捕获错误“）发生时都调用”uncaught“处理函数。

另一个更常见的建议是Promise应该添加个`done(..)`方法，本质上是标记Promise链”结束了“。`done(..)`不会创建也不会返回一个Promise，因此传入`done(..)`中的回调不会向一个不存在的链式Promise报告问题。

那么会发生什么呢？在未捕获错误（uncaught error）情况下，它会按照你预期的那样得到处理：`done(..)`内的rejection处理函数中的任何异常，都会被当做全局的未捕获错误抛出（通常在开发者控制台）：

```javascript
var p = Promise.resolve( 42 );

p.then(
    function fulfilled(msg){
        // numbers don't have string functions,
        // so will throw an error
        console.log( msg.toLowerCase() );
    }
)
.done( null, handleErrors );

// if `handleErrors(..)` caused its own exception, it would
// be thrown globally here
```

似乎听起来比永不终止的链或者随机超时更具吸引力。但最大的问题是它并不是ES6标准的一部分，因此不论听起来有多好，离称为一个可信赖并且普遍的解决方案还有更长的路。

那么我们就这么被困住了吗？不全是。

浏览器有一个我们代码不具备的独特能力：它们能够追踪并且很确定的知道任一个对象被丢弃和垃圾回收的时间。因此，浏览器可以追踪Promise对象，一旦被垃圾回收，如果其中有一个rejection，浏览器能很确定的知道这是个正当的”未捕获错误“，并且很自信地将其报告给开发者控制台。


**注意：** 写到这时，Chrome和Firefox在此种”uncaught rejection“能力方面有一些早期的尝试，尽管支持的不完全。

然而，如果一个Promise没有被垃圾回收--通过各种不同的编程模式，这种情况很容易发生--浏览器的垃圾回收嗅探就无法帮你诊断出有一个静默的rejected Promise。

还有其它办法吗？是的。

### 成功之坑（Pit of Success）

关于Promise的行为将来可能变成什么样，接下来所说的只是理论性的。我认为这远远优于我们当前所拥有的。并且我认为这种改变在后ES6中是可能的，因为它不会打破浏览器对ES6 Promise的兼容。此外，如果细心一点的话，这种改变可以polyfill，让我们看下：

+ 如果在那个时刻Promise上没有注册错误处理函数，Promise能够在下一个作业（Job）或者事件轮询tick时，默认地（向开发者控制台）报告任何rejection。
+ 你想让一个rejected Promise在被监听前的一段不确定时间内保持rejected状态，此时你可以调用`defer()`，它可以阻止该Promise的自动错误上报。

如果一个Promise被rejected了，它会默认将之上报给开发者控制台（而不是默认静默）。你可以选择隐式地（在rejection前注册一个错误处理函数）或者显式地（采用`defer()`）退出错误上报。无论哪一种情况，都是由你控制误报（false positives）。

考虑如下：

```javascript
var p = Promise.reject( "Oops" ).defer();

// `foo(..)` is Promise-aware
foo( 42 )
.then(
    function fulfilled(){
        return p;
    },
    function rejected(err){
        // handle `foo(..)` error
    }
);
...
```

当我们创建`p`时，我们打算等一会再使用/监听它的rejection，因此我们调用了`defer()`。--因此没有全局上报。为实现链式，`defer()`只是简单地返回同样的promise。

`foo(..)`返回的promise立刻附上了一个错误处理函数，因此隐式地选择退出，并且也没有错误上报。

但`then(..)`调用返回的promise没有附上`defer(..)`或者错误处理函数，因此，如果它reject（因内部的任一个解析处理函数），就会被当作未捕获异常上报至开发者控制台。

**这种设计就是成功之坑。** 默认情况下，所有错误，都会被处理或者上报--这是多数开发者在绝大多数情况下希望的那样。你既可以注册一个处理函数，也可以选择退出，表明你想推迟到以后处理异常，只在那种特定情况下你选择额外的责任（译者注：指自己处理异常）。

这种方法唯一的危险就是，如果你`defer()`一个Promise，但之后无法监听/处理rejection。

但是你必须主动调用`defer()`来选择进入绝望深渊--默认是成功之坑--因此，对于你自己的错误，我们所能做的不多。

我认为Promise错误处理仍然有希望（后ES6）。我希望当权者（译者注：此处指ES规范的制定者）重新思考下这种情况并且考虑这个方案。同时，你可以自己实现（对读者而言是个不小的挑战！），或者使用一个更精简的库！

**注意：** 错误处理/上报的精确模型在我的*异步队列*Promise 抽象库中实现了，会在本书的附录A中讨论。

## Promise模式（Promise Patterns）

我们已经见识了采用Promise链（this-then-this-then-that流控制）实现的序列模式，但在Promise外，构建在异步模式上的抽象，还有许多变体。这些模式用来简化异步流控制的表示--这使得我们的代码更合理，更易于维护--即使是在程序中最复杂的部分。

有两种这样的模式直接被编进了原生ES6 `Promise`实现中，因此我们可以很方便地使用它们，来作为其它模式的构建块。

### Promise.all([])

在异步序列中（Promise链），在任一给定时刻只能协调一个异步任务--step 2严格地跟在step 1后面，step 3严格地跟在step 2后面。但要是同时进行两步或更多步呢（即”并行“）？

在传统编程术语中，”门（gate）“机制是指在继续之前，需要等待两个或更多的并行/并发任务完成。完成的顺序并不重要，只需要它们都完成，进而打开门，让流控制通过。

在Promise API中，我们称这种模式为`all([ .. ])`。

假设你想同时发两个Ajax请求，并且在发第三个请求前，需要等到两个都完成，顺序不重要。考虑如下：

```javascript
/ `request(..)` is a Promise-aware Ajax utility,
// like we defined earlier in the chapter

var p1 = request( "http://some.url.1/" );
var p2 = request( "http://some.url.2/" );

Promise.all( [p1,p2] )
.then( function(msgs){
    // both `p1` and `p2` fulfill and pass in
    // their messages here
    return request(
        "http://some.url.3/?v=" + msgs.join(",")
    );
} )
.then( function(msg){
    console.log( msg );
} );
```

`Promise.all([ .. ])`接收单个参数，即一个`array`，通常由Promise实例组成。`Promise.all([ .. ])`返回的promise会接收一个fulfillment信号（代码中的`msgs`），它是一个由所有传入的promise生成的fulfillment信号组成的`array`，顺序与指定的一致（与达到fulfillment状态的时间顺序无关）。

**注意：** 从技术来讲，传入`Promise.all([ .. ])`的`array`值可以是Promise、thenable或者甚至是立即（immediate）值。每一个值都会传入到`Promise.resolve(..)`中，确保最终都是真正的Promise，因此立即值会以该值被标准化为Promise。如果`array`是空的
，则主Promise会立即fulfilled。

当且仅当所有的成员promise都fulfilled时，`Promise.all([ .. ])`返回的主promise才fulfilled。如果任一个promise被rejected了，主`Promise.all([ .. ])`promise立即被rejected，不管其它promise的结果。

记住，在每个promise后都要附个rejection/error处理函数，尤其是`Promise.all([ .. ])`返回的promise。

### Promise.race([])

尽管`Promise.all([ .. ])`能够同时协调多个Promise，并且假定所有的promise都需要fulfillment，但有时你只想响应”第一个跨过终点线的Promise“，而让其它Promise离开。

这种模式通常称为”闩（latch）“，但在Promise中，称为'race'。

**警告：** ”只有第一个跨过终点线的赢“，尽管这一比喻很恰当，但是，"race"有点被过度使用了，因为在程序中，”竞态（race condition）“通常被认为是bug（见第一章）。不要把`Promise.race([ .. ])`和”race condition“弄混淆了。


`Promise.race([ .. ])`也接收单个`array`参数，包括一个或多个Promise、thenable或者立即值。包含立即值并没有太多的实际意义，因为第一个列出来的立即值很明显会赢--就像赛跑中，一个站在终点线开始跑的人一样！

类似于`Promise.all([ .. ])`，当任一个Promise解析是fulfillment时，`Promise.race([ .. ])`即fulfill，当任一个Promise解析是rejection时，`Promise.race([ .. ])`即reject。

**警告：** 一个”race“至少需要一个”runner“，因此，如果你传入一个空`array`，主`race([..])`Promise永远不会解析。这是个footgun（译者注：没查到这是个什么鬼）！ES6本该指定它要么fulfill，要么reject，或者抛出某种同步的错误。不幸的是，由于Promise库早于ES6 `Promise`，他们只好将之放至一边，因此千万不要传一个空`array`。

让我们重新看一下前面的并发Ajax例子，但是以`p1`和`p2`竞争的形式：

```javascript
// `request(..)` is a Promise-aware Ajax utility,
// like we defined earlier in the chapter

var p1 = request( "http://some.url.1/" );
var p2 = request( "http://some.url.2/" );

Promise.race( [p1,p2] )
.then( function(msg){
    // either `p1` or `p2` will win the race
    return request(
        "http://some.url.3/?v=" + msg
    );
} )
.then( function(msg){
    console.log( msg );
} );
```

因为只有一个promise胜出，所以fulfillment值是单个信息值，不是如`Promise.all([ .. ])`的`array`。

#### 超时竞赛（Timeout Race）

我们早先看过这个例子，用来说明如何使用`Promise.race([ .. ])`表示”promise timeout“模式：

```javascript
// `foo()` is a Promise-aware function

// `timeoutPromise(..)`, defined ealier, returns
// a Promise that rejects after a specified delay

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

多数情况下，这种超时模式很管用。但有些细节需要考虑，坦白来说，这些细节对`Promise.race([ .. ])`和`Promise.all([ .. ])`同等适用。

#### ”最后“（”Finally“）

关键问题是，”被废弃/忽略的promise发生了什么？“我们不是从性能角度来问的--它们通常最终会被垃圾回收--而是从行为角度（副作用等等）。Promise不能取消--也不应该取消，否则会破坏外部不可变性信任，这会在本章的”Promise Uncancelable“一节中讨论--因此，只能静默忽略这些promise。

但要是前面例子中的`foo()`保留着某种资源，但是超时首先触发，造成那个promise被忽略了呢？在超时后，这种模式能够主动释放保留的资源？或者取消可能造成的任何副作用吗？要是你想要的只是记录`foo()`超时呢？

一些开发者已经提议，Promise需要一个`finally(..)`回调注册，总是在Promise解析完后调用，允许你指定任何必要的清理工作。目前并不存在于规范中，但可能出现在ES7+中。让我们拭目以待。

它看起来可能会是这样：

```javascript
var p = Promise.resolve( 42 );

p.then( something )
.finally( cleanup )
.then( another )
.finally( cleanup );
```

**注意：** 在各种Promise库中，`finally(..)`仍然会创建并返回一个新的Promise（为了保证链式）。如果`cleanup(..)`函数返回一个Promise，它会被链接到链中，这意味着你仍然可能有我们之前讨论的未处理rejection问题。

同时，我们可以实现个静态辅助函数，允许我们监听（不会影响）Promise的解析结果：

```javascript
// polyfill-safe guard check
if (!Promise.observe) {
    Promise.observe = function(pr,cb) {
        // side-observe `pr`'s resolution
        pr.then(
            function fulfilled(msg){
                // schedule callback async (as Job)
                Promise.resolve( msg ).then( cb );
            },
            function rejected(err){
                // schedule callback async (as Job)
                Promise.resolve( err ).then( cb );
            }
        );

        // return original promise
        return pr;
    };
}
```

在之前超时例子中使用该监听函数：

```javascript
Promise.race( [
    Promise.observe(
        foo(),                  // attempt `foo()`
        function cleanup(msg){
            // clean up after `foo()`, even if it
            // didn't finish before the timeout
        }
    ),
    timeoutPromise( 3000 )  // give it 3 seconds
] )
```

`Promise.observe(..)`辅助函数只是为了说明如何在不影响它们的情况下，监听Promise的完成。其它Promise库有自己的解决方案。无论你怎么实现，都要确保你的Promise不会意外地被静默忽略。

### all([..])和race([..])的变体（Variations on all([ .. ]) and race([ .. ])）

尽管原生ES6只有内建的`Promise.all([ .. ])`和`Promise.race([ .. ])`，但基于这些，还可以实现一些常用的模式：

+ `none([..])`有点像`all([ .. ])`，但是fulfillment和rejection是相反的，所有的Promise都应当被rejected--rejection变为fulfillment值，反之亦然。
+ `any([ .. ]) `有点像`all([ .. ])`，但是它会忽略任何rejection，因此只需要一个fulfill，而不是全部。
+ 'first([ .. ])'有点像`any([ .. ])`的竞争版，即忽略任何rejection，一旦第一个Promise fulfill，该Promise就fulfill了。
+ `last([ .. ])`有点像'first([ .. ])'，但是最后一个Promise获胜。

有些Promise抽象库实现了以上这些，但是你也可以利用Promise的机制（`race([ .. ])`和`all([ .. ])`）自己定义。

例如，这是我们定义的`first([..])`:

```javascript
// polyfill-safe guard check
if (!Promise.first) {
    Promise.first = function(prs) {
        return new Promise( function(resolve,reject){
            // loop through all promises
            prs.forEach( function(pr){
                // normalize the value
                Promise.resolve( pr )
                // whichever one fulfills first wins, and
                // gets to resolve the main promise
                .then( resolve );
            } );
        } );
    };
}
```

**注意：** 如果所有promise都reject，这个`first(..)`实现中并没有reject，只是简单的挂起，就像`Promise.race([])`一样。如果愿意，你可以另外添加逻辑来跟踪每个promise 的rejection，如果所有promise均reject，在主promise上调用`reject()`。我们将之留给读者作为练习。

### 并行遍历（Concurrent Iterations）

有时，你想遍历一列Promise，并针对所有这些Promise执行某些任务，就像对同步的`array`一样（比如，`forEach(..)`，`map(..)`，`some(..)`和`every(..)`）。如果对每个promise执行的任务是严格同步的，这几个方法就可以了，正如我们之前代码中用到的`forEach(..)`一样。

但如果任务是异步的，或者应该并发执行，那么你可以使用库提供的这些utility的异步版本。

例如，考虑一个异步的`map(..)`utility，它接受一个`array`值（可能是Promise，也可能是其它）和一个针对每个值的执行函数（任务）。`map(..)`函数本身返回一个promise，它的fulfillment值是一个`array`，保存着（以同样的映射顺序）每个任务返回的fulfillment值：

```javascript
if (!Promise.map) {
    Promise.map = function(vals,cb) {
        // new promise that waits for all mapped promises
        return Promise.all(
            // note: regular array `map(..)`, turns
            // the array of values into an array of
            // promises
            vals.map( function(val){
                // replace `val` with a new promise that
                // resolves after `val` is async mapped
                return new Promise( function(resolve){
                    cb( val, resolve );
                } );
            } )
        );
    };
}
```
**注意：** 在这个`map(..)`实现中，你无法发出异步rejection信号，但是如果在映射回调（`cb(..)`）内部发生同步异常/错误，`Promise.map(..)`返回的主promise就会reject。

让我们举例说明一下多个Promise（而不是简单值）时的`map(..)`用法：

```javascript
var p1 = Promise.resolve( 21 );
var p2 = Promise.resolve( 42 );
var p3 = Promise.reject( "Oops" );

// double values in list even if they're in
// Promises
Promise.map( [p1,p2,p3], function(pr,done){
    // make sure the item itself is a Promise
    Promise.resolve( pr )
    .then(
        // extract value as `v`
        function(v){
            // map fulfillment `v` to new value
            done( v * 2 );
        },
        // or, map to promise rejection message
        done
    );
} )
.then( function(vals){
    console.log( vals );    // [42,84,"Oops"]
} );
```

## Promise API 回顾（Promise API Recap）
-----

让我们回顾一下本章中零零碎碎展开的ES6 `Promise` API。

**注意：** 下面的API只是原生ES6才有的，但仍有一些兼容规范的polyfill（不只是简单地扩展Promise库），它们可以定义`Promise`及其所有相关行为，以便于在pre-ES6（ES6前）的浏览器中使用原生的Promise。其中一个polyfill是“Native Promise Only”（[http://github.com/getify/native-promise-only](http://github.com/getify/native-promise-only)），我写的！

### new Promise(..) Constructor

*暴露构造函数（ revealing constructor ）*（见 **Promise "Events"** 一节）必须配合`new`使用，必须提供一个同步/立即调用的回调。这个函数传入两个回调充当promise的解析功能。我们通常将它们标为`resolve(..)`和`reject(..)`：

```javascript
var p = new Promise( function(resolve,reject){
    // `resolve(..)` to resolve/fulfill the promise
    // `reject(..)` to reject the promise
} );
```

`reject(..)`只是简单地reject该promise，但`resolve(..)`根据传入的值，既可以fulfill该promise，也可以reject该promise。如果传入`resolve(..)`的是个立即的，非Promise，非thenable值，之后就会以该值fulfill该promise。

但是如果`resolve(..)`传入的是个真正的Promise或者thenable值，那么该值就会被递归拆析，promise会接收最终的解析结果/状态。

### Promise.resolve(..) and Promise.reject(..)

创建一个已经rejected的Promise的简写形式为`Promise.reject(..)`，因此这两个promise是等价的：

```javascript
var p1 = new Promise( function(resolve,reject){
    reject( "Oops" );
} );

var p2 = Promise.reject( "Oops" );
```

类似于`Promise.reject(..)`，`Promise.resolve(..)`通常用来创建一个已经fulfilled的Promise。然而，`Promise.resolve(..)`同样也拆析thenable值（正如多次讨论的那样）。在那种情况下，返回的Promise接收传入的thenable的最终解析结果，既可能是fulfillment，又可能是rejection：

```javascript
var fulfilledTh = {
    then: function(cb) { cb( 42 ); }
};
var rejectedTh = {
    then: function(cb,errCb) {
        errCb( "Oops" );
    }
};

var p1 = Promise.resolve( fulfilledTh );
var p2 = Promise.resolve( rejectedTh );

// `p1` will be a fulfilled promise
// `p2` will be a rejected promise
```

记住，如果你传入一个真正的Promise，`Promise.resolve(..)`什么也不会做；它只会直接返回该值。因此如果恰巧是个真正的Promise，而你不知道是何种值时，调用`Promise.resolve(..)`不会有开销。

### then(..) and catch(..)

每个Promise实例（**不是** `Promise` API命名空间）都有`then(..)`和`catch(..)`方法，允许为Promise注册fulfillment和rejection处理函数。一旦Promise解析完，其中一个就会被调用，并且是异步调用的（见第一章的"Jobs"）。

`then(..)`接收一个或两个参数，第一个作为fulfillment回调，第二个作为rejection回调。如果省略任一个或者传入一个非函数值，就会用相应的默认函数代替。默认的fulfillment回调只是简单的传递信息，而默认的rejection回调简单地重新抛出（传播）接收到的错误原因。

`catch(..)`只接收rejection回调作为参数，会自动替换上默认fulfillment回调。换句话说，等价于`then(null,..)`：

```javascript
p.then( fulfilled );

p.then( fulfilled, rejected );

p.catch( rejected ); // or `p.then( null, rejected )`
```

`then(..)`和`catch(..)`同样会创建并返回一个新的promise，可用来表示Promise的链式流控制。

如果fulfillment或者rejection回调有异常抛出，则返回的promise就被reject了。如果任一个回调返回一个立即的，非Promise，非thenable值，则那个值就被设为返回promise的fulfillment。如果fulfillment处理函数指定返回了一个promise或者thenable值，则那个值会被拆析并成为返回promise的解析结果。

### Promise.all([ .. ]) and Promise.race([ .. ])

ES6 `Promise` API中的静态函数`Promise.all([ .. ])`和`Promise.race([ .. ])`，都创建了一个Promise作为返回值。那个promise的解析结果完全决定于你传入的promise数组。

对于`Promise.all([ .. ])`而言，若要返回的promise fulfill，则传入的所有promise必须都fulfill。如果任一个promise被reject了，则主promise也立即被reject（忽略其它promise的结果）。对于fulfillment而言，你传入的`array`必须都是fulfillment的promise。对于rejection而言，只要有一个rejection的promise即可。这种模式通常称为“门（gate）”：在门打开前，所有人必须都到场。

对于`Promise.race([ .. ])`，只有第一个解析的promise（fulfillment或者rejection）“胜出”，解析结果无论是什么，都会成为返回的promise的解析结果。这种模式通常称为“闩（latch）”：第一个打开闩的人通过。考虑如下代码：

```javascript
var p1 = Promise.resolve( 42 );
var p2 = Promise.resolve( "Hello World" );
var p3 = Promise.reject( "Oops" );

Promise.race( [p1,p2,p3] )
.then( function(msg){
    console.log( msg );     // 42
} );

Promise.all( [p1,p2,p3] )
.catch( function(err){
    console.error( err );   // "Oops"
} );

Promise.all( [p1,p2] )
.then( function(msgs){
    console.log( msgs );    // [42,"Hello World"]
} );
```

**警告：** 如果空`array`传入`Promise.all([ .. ])`，则立即fulfill，但是`Promise.race([ .. ])`会永远挂起，从不解析。

ES6 的`Promise`相当简单直接。基本上足够满足绝大多数异步需求了，在重排代码，使之从地狱回调转向其它更好的方式时，是个不错的选择。

但是在某些应用中，有些复杂的异步需求，Promise处理的不是很好。在下一节，我们将会仔细探究这些局限性，并以此作为Promise库跟进的动力。

## Promise的局限（Promise Limitations）

本节讨论的所有细节在本章中都略微提及了，让我们专门回顾一下这些不足。

### 序列化错误处理（Sequence Error Handling）

本章早些时候，我们已经仔细讨论过Promise式的错误处理了。Promise设计方式的不足--尤其是成链的方式--很容易造成陷阱，即Promise链中的错误可能被静默忽略。

但是，关于Promise 错误，还需要考虑其它一些东西。因为Promise链只是将成员promise连在一起，没有实体（entity）能够将整个链视作单个个体，这意味着没有外部方法能够监听可能发生的错误。

如果你创建了一个没有错误处理的Promise链，那么链中的任何一个错误都会沿着链无限传播下去，直至被监听（通过在某步注册rejection处理函数）。因此，在那种特定情况下，有一个指向链中的最后一个promise就行了（以下代码中是`p`）的引用，因为可以在那注册一个rejection处理函数，如果有任何错误传过来，就会被通知到。

```javascript
// `foo(..)`, `STEP2(..)` and `STEP3(..)` are
// all promise-aware utilities

var p = foo( 42 )
.then( STEP2 )
.then( STEP3 );
```

尽管看起来有点迷糊，此处的`p`并不指向链中的第一个promise(`foo(42)`调用返回的)，而是指向最后一个promise，`then(STEP3)`调用返回的。

另外，该promise链中没有一步监听错误处理，意味着你可以在`p`上注册一个rejection错误处理函数，如果链中发生任何错误，rejection注册函数就会被通知到：

```javascript
p.catch( handleErrors );
```

但是如果链中的每一步有自己的错误处理函数（或许被隐藏/抽象，你看不到），你的`handleErrors(..)`就不会被通知到。这可能是你想要的--毕竟，它是一个“rejection处理函数”--也可能不是你想要的。完全丧失接收通知的能力是个不足之处，在某些情况下限制了功能实现。

这一不足和现有的`try..catch`是一样的，它能够捕获异常并简单地掩盖异常。因此，这不是**Promise独有的**不足，但我们希望有一些解决方案。

不幸的是，在Promise链中，无法保持对中间步骤的引用，因此，没有这些引用的话，就无法附上错误处理函数来可靠地监听错误。

### 单个值（Single Value）

从定义上来说，Promise只有单个fulfillment值或者rejection原因短语，在简单例子中，这不是个大问题。但在更复杂的场景中，你就会觉得捉襟见肘了。

通常的建议是构建一个值包装器（比如一个`object`或者`array`）来包含多个值。这种方法有效，但是在每步中包装、拆析信息相当别扭和繁琐。

#### 分离值（Splitting Values）

有时你可以把这当做一个信号，即应该将问题分解为多个Promise。

假设有一个utility`foo(..)`，它异步生成两个值（`x`和`y`）。

```javascript
function getY(x) {
    return new Promise( function(resolve,reject){
        setTimeout( function(){
            resolve( (3 * x) - 1 );
        }, 100 );
    } );
}

function foo(bar,baz) {
    var x = bar * baz;

    return getY( x )
    .then( function(y){
        // wrap both values into container
        return [x,y];
    } );
}

foo( 10, 20 )
.then( function(msgs){
    var x = msgs[0];
    var y = msgs[1];

    console.log( x, y );    // 200 599
} );
```

首先，让我们重排一下`foo(..)`的返回值，以便我们在传递值时，不需要将`x`和`y`包进一个`array`中。可以将每个值包到自己的promise中：

```javascript
function foo(bar,baz) {
    var x = bar * baz;

    // return both promises
    return [
        Promise.resolve( x ),
        getY( x )
    ];
}

Promise.all(
    foo( 10, 20 )
)
.then( function(msgs){
    var x = msgs[0];
    var y = msgs[1];

    console.log( x, y );
} );
```

一个promise `array`真的比通过单个promise传递的`array`值好吗？从句法结构上来看，并没有多少改善。

但这种方式更符合Promise的设计理念。将`x`和`y`的计算分到不同的函数中去，这样将来更容易重构。

让调用代码决定如何安排这两个promise，这种方式更清晰灵活--此处采用`Promise.all([ .. ])`，但当然不是唯一的选项--而不是把`foo(..)`内的细节抽离出来。

#### 打开/展开参数（Unwrap/Spread Arguments）

`var x = ..`和`var y = ..`赋值操作仍然是很别扭的开销。我们可以在辅助函数中采用一些小的功能性手段（致敬Reginald Braithwaite, @raganwald on Twitter）：

```javascript
function spread(fn) {
    return Function.apply.bind( fn, null );
}

Promise.all(
    foo( 10, 20 )
)
.then(
    spread( function(x,y){
        console.log( x, y );    // 200 599
    } )
)
```

这种更好一点！当然，你可以写成内联函数样式，避免额外的辅助函数：

```javascript
Promise.all(
    foo( 10, 20 )
)
.then( Function.apply.bind(
    function(x,y){
        console.log( x, y );    // 200 599
    },
    null
) );
```

这种把戏可能很简洁，但是ES6有个更好的方案：解构。数组解构赋值的形式看起来是这样子的：

```javascript
Promise.all(
    foo( 10, 20 )
)
.then( function(msgs){
    var [x,y] = msgs;

    console.log( x, y );    // 200 599
} );
```

但最好的是，ES6提供数组参数解构形式：

```javascript
Promise.all(
    foo( 10, 20 )
)
.then( function([x,y]){
    console.log( x, y );    // 200 599
} );
```

现在，我们已经奉行了一个Promise一个值（one-value-per-Promise）的准则，但同时将支持样板减到最少！

**注意：** 要想了解更多关于ES6解构形式的信息，请看本系列的*ES6 & Beyond*。

### 单次解析（Single Resolution）

Promise最固有的行为之一就是Promise只能被解析一次（fulfillment或者rejection）。对于许多异步使用场景而言，你只需获取一次值，因此这种形式效果很好。

但有许多异步场景符合另一种不同的模型--即更类似于事件或者数据流。从表面上看，就算可以，也并不清楚Promise能够在多大程度上适用于这些使用场景。在Promise之外，没有重大的抽象，因而完全缺乏处理多次值解析的能力。

假设有个场景，为响应一个激励（比如一个事件），它可能发生很多次，比如按钮点击，你想触发一系列的异步操作。

这可能不是你想要的方式：

```javascript
// `click(..)` binds the `"click"` event to a DOM element
// `request(..)` is the previously defined Promise-aware Ajax

var p = new Promise( function(resolve,reject){
    click( "#mybtn", resolve );
} );

p.then( function(evt){
    var btnID = evt.currentTarget.id;
    return request( "http://some.url.1/?id=" + btnID );
} )
.then( function(text){
    console.log( text );
} );
```

这段代码只在你的应用要求按钮只点击一次时有效。如果再点一次按钮，`p` promise已经解析了，因此第二次`resolve(..)`调用就会被忽略。

反而，你可能需要转换一下方案，每次事件触发时创建一个全新的Promise链。

```javascript
click( "#mybtn", function(evt){
    var btnID = evt.currentTarget.id;

    request( "http://some.url.1/?id=" + btnID )
    .then( function(text){
        console.log( text );
    } );
} );
```

这种方法就会奏效，因为按钮的每一次`"click"`事件都会生成一个全新的Promise序列。

但是，在事件处理函数中定义整个Promise链，除了难看以外，从某种角度来说，这种设计违反了关注/功能分离（separation of concerns/capabilities，SoC）的理念。你可能想在代码的其它地方定义事件处理函数，不同于定义事件响应（Promise链）所在的位置。没采用辅助机制的话，这种模式相当别扭。

**注意：** 另一种阐述这种局限的方式是，要是我们能构建一种让Promise链订阅的“监听器（observable）”就好了。已经有库实现了这些抽象（比如RxJS--[http://rxjs.codeplex.com/](http://rxjs.codeplex.com/)），但是这些抽象很臃肿，以至于你再也看不清Promise的本质了。这些臃肿的抽象带来了一些需要注意的问题，比如这些机制（不包括Promise）是否和Promise设计的那样*值得信赖*。我们会在附录B中再讨论“Observable”模式。

### 惯性（Inertia）

在代码中开始使用Promise的一个具体障碍是，现存的代码都不是基于Promise（Promises-aware ）。如果有许多基于回调的代码，那么以同样方式编程更容易。

“运转中（采用回调）的代码库会继续运转（采用回调），除非聪明的、具有Promise意识的开发者采取行动”。

Promise提供了一种不同的模式，正因如此，编码方式可能有些不同，某些情况下，完全不同。你必须刻意这样做，因为Promise本身无法将你从早已习惯的编码方式中脱离出来。

考虑以下一个基于回调的场景：

```javascript
function foo(x,y,cb) {
    ajax(
        "http://some.url.1/?x=" + x + "&y=" + y,
        cb
    );
}

foo( 11, 31, function(err,text) {
    if (err) {
        console.error( err );
    }
    else {
        console.log( text );
    }
} );
```

是否立刻就能看出第一步应该干什么，即如何将这个基于回调的代码转为基于Promise的代码？取决于你的经验。你使用Promise的实践越多，就会越觉得自然。Promise并没有宣称具体该怎么做--没有放之四海皆准的答案--因此责任在你。

正如我们之前讨论过的一样，我们确实需要一个基于Promise，而不是回调的Ajax utility，我们可以称之为`request(..)`。你可以像我们一样实现自己的方法。但是为每个基于回调的utility手动定义Promise式的包装器开销很大，你就更不会选择基于Promise的重构了。

对于这一不足，Promise没有直接的答案。然而，绝大多数Promise库确实提供这样的一个辅助函数。但就算没有这样的库，辅助函数也可能像这样：

```javascript
// polyfill-safe guard check
if (!Promise.wrap) {
    Promise.wrap = function(fn) {
        return function() {
            var args = [].slice.call( arguments );

            return new Promise( function(resolve,reject){
                fn.apply(
                    null,
                    args.concat( function(err,v){
                        if (err) {
                            reject( err );
                        }
                        else {
                            resolve( v );
                        }
                    } )
                );
            } );
        };
    };
}
```

OK，这仅仅是个小小的实验程序。然而，尽管看起来有点吓人，但它并不是你想得那么糟。它接收一个函数，该函数期望一个错误优先式的回调作为最后一个参数，返回一个自动创建并返回promise的新函数
，通过将其连到Promise fulfillment/rejection上来替换掉你的回调函数。

不要再浪费时间讨论`Promise.wrap(..)`辅助函数是如何工作的，让我们看下如何使用吧：

```javascript
var request = Promise.wrap( ajax );

request( "http://some.url.1/" )
.then( .. )
..
```

哇哦，相当简单！

`Promise.wrap(..)`并不生成一个Promise。它返回一个能生成promise的函数。从某种意义上来说，Promise生成函数可以被视作“Promise工厂”。我提议采用“promisory”来为其命名（"Promise" + "factory"）。

将一个期望回调的函数包装成一个Promise式的函数的行为有时称为“提升（lifting）”或者“promise化（promisifying）”。但对于结果函数，除了叫“提升函数（lifted function）”，似乎没有一个标准的术语来称呼它，因此我更喜欢“promisory”，因为它更具描述性。

**注意：** Promisory并不是个拼凑的术语。它是个真正的词，定义是包含或者传递一个promise。那正是这些函数所做的，因此这是一个完美匹配的术语！

因此，`Promise.wrap(ajax)`生成了称为`request(..)`的`ajax(..)` promisory，那个promisory为Ajax响应生成Promise。

那么回到早先的例子，我们需要为`ajax(..)`和`foo(..)`创建promisory：

```javascript
// make a promisory for `ajax(..)`
var request = Promise.wrap( ajax );

// refactor `foo(..)`, but keep it externally
// callback-based for compatibility with other
// parts of the code for now -- only use
// `request(..)`'s promise internally.
function foo(x,y,cb) {
    request(
        "http://some.url.1/?x=" + x + "&y=" + y
    )
    .then(
        function fulfilled(text){
            cb( null, text );
        },
        cb
    );
}

// now, for this code's purposes, make a
// promisory for `foo(..)`
var betterFoo = Promise.wrap( foo );

// and use the promisory
betterFoo( 11, 31 )
.then(
    function fulfilled(text){
        console.log( text );
    },
    function rejected(err){
        console.error( err );
    }
);
```

当然，尽管我们重构了`foo(..)`来使用新的`request(..)` promisory，但是我们也可以只将`foo(..)`本身变成一个promisory ，而不是保持基于回调的状态并且需要创建和使用随后的`betterFoo(..)` promisory。这仅取决于`foo(..)`是否需要保持在基于回调的状态来兼容代码库的其它部分。

考虑如下代码：

```javascript
// `foo(..)` is now also a promisory because it
// delegates to the `request(..)` promisory
function foo(x,y) {
    return request(
        "http://some.url.1/?x=" + x + "&y=" + y
    );
}

foo( 11, 31 )
.then( .. )
..
```

尽管ES6 Promise没有原生提供诸如promisory包装的辅助函数，但绝大多数库提供，或者也可自己实现。不论哪一种，Promise的这一不足可以在没有太大痛苦（当然是相比于地狱回调的痛苦）的情况下得到解决。

### Promise不可取消（Promise Uncancelable）

一旦创建了一个Promise并为其注册一个fulfillment和/或rejection处理函数，如果由于某些其它原因，使得任务突然变得无意义了，你无法从外部阻止信息的传播。

**注意：** 许多Promise抽象库提供工具来取消Promise，但这是个很糟糕的主意！许多开发者希望Promise要是原生就具有外部取消功能就好了，但问题是这会让一个Promise 解析者/监听者影响其它解析者对同一个Promise的的监听。这违背了未来值可信原则（即外部不可变性），更是“超距作用（action at a distance）”反模式（anti-pattern）（[http://en.wikipedia.org/wiki/Action_at_a_distance_%28computer_programming%29](http://en.wikipedia.org/wiki/Action_at_a_distance_%28computer_programming%29)）的具体体现。不管看起来如何有用，它会使你直接回到回调一样的噩梦中。

考虑早先的超时Promise场景：

```javascript
var p = foo( 42 );

Promise.race( [
    p,
    timeoutPromise( 3000 )
] )
.then(
    doSomething,
    handleError
);

p.then( function(){
    // still happens even in the timeout case :(
} );
```

"超时"是在promise `p`的外部的，因此超时后，`p`本身会继续运行，这不是我们想要的。

一种选择是侵入性地（invasively）定义解析回调：

```javascript
var OK = true;

var p = foo( 42 );

Promise.race( [
    p,
    timeoutPromise( 3000 )
    .catch( function(err){
        OK = false;
        throw err;
    } )
] )
.then(
    doSomething,
    handleError
);

p.then( function(){
    if (OK) {
        // only happens if no timeout! :)
    }
} );
```

这很难看，虽然有效，但是远不理想。一般来说，需要避免这种情况。

但如果不这么做的话，这种丑陋的办法会让你认识到，*可取消性（cancelation）*是个在Promise之外的，一种属于更高级抽象的功能。我建议你转向Promise抽象库寻求帮助，而不是自己实现。

**注意：** 我的“异步队列（asynquence）”Promise抽象库正好提供了这样一个抽象和队列的`abort()`功能。将会在附录A中讨论。

单个Promise并不是真正的流控制机制（至少从是否有意义方面来说不是），这正是*可取消性（cancelation）*所指的那样；这就是Promise 可取消性让人觉得别扭的原因。

与此相反，多个Promise链在一起--我称为一个“序列”--才是流控制的表示形式，因此，在这个抽象层面定义可取消性才比较合适。

单个Promise不应该被取消，但取消*序列（sequence）*是有意义的，因为你不会像Promise那样将序列当做单个不可变值传递。

### Promise性能（Promise Performance）

这一局限既简单又复杂。

基本的基于回调的异步任务链对比Promise链，各有多少代码运行，相比于此，很清楚的一点是Promise运行的相对多一点，这意味着Promise天生就慢一点。简单回想下Promise提供的信任保证，相比于为了达到同样的效果，不得不在回调之外再布一层专门的解决方案代码。

需要做更多工作，更多防护，这意味着相比于裸奔的、不可信任的回调，Promise更慢一些。这很明显，很简单就能想到。

但是慢多少呢？呃。。很难全面地回答这个难题。

坦白来说，这就像是苹果和橘子的比较，因此问的问题可能就是错的。你应该问的是，手动部署同样效果防御代码的回调，是否比Promise实现更快些。

如果Promise有合理的性能局限，更多的在于Promise不提供是否需要信任保护的选项--Promise总是提供信任保护。

然而，如果我们承认Promise通常比对等的non-Promise、non-trustable回调*稍微慢一点*--假设某些地方你觉得缺乏信任是合理的--那不就意味着应当完全避免使用Promise，犹如你的整个应用完全由尽可能快的代码驱动吗？

总结一下：如果你的代码是那样的话，那么**JavaScript是完成这些任务的恰当语言吗？**可以优化JavaScript，使之能够高性能地运行应用（见第五、六章）。但是，鉴于提供的种种好处，沉迷于Promise的一点点性能折中，真的合适吗？

另一个微妙的问题是，Promise*让一切都变成异步了*，这意味着一些立即（同步）完成的步骤的下步进展仍然被推迟到Job中（见第一章）。也就是说很可能一系列Promise任务可能比相同的采用回调串联起来的序列完成的慢。

当然，此处的问题是：这些性能方面的潜在不足真的*抵得上*整篇文章所提到的Promise的优点吗？

在我看来，所有你认为Promise性能不好的情况，实际上都是一种反模式
，即通过避免使用Promise来优化掉Promise的可信任和可组合的优点。

反而，你应该在整个代码库中默认使用Promise，之后简述和分析应用的热（重要）路径（译者注：指应用中使用频率最高的部分）。Promise*真的*是性能瓶颈，或者只是理论上的性能低下？只有之后，有了有效地基准（见第六章），才能谨慎负责地在确定的关键领域中评价Promise。

Promise有点慢，但作为交换，你获得了信任、非Zalgo可预测性和构建的可组合性。或许缺陷不是性能，而是你对Promise优点的缺乏认识？

## 回顾（Review）

Promise很棒，使用它吧。它解决了*控制权反转*问题，这个问题在只有回调的代码中一直困扰我们。

Promise并没有舍弃回调，只是将那些回调重新编排，成为一个站在我们和第三方实用程序间可信任的中间机制。

Promise也开始以序列化的方式（尽管并不完美）更好地表示异步流，这能帮助我们的大脑更好地规划和维护异步JS代码。我们会在下一章看到一个更好的表示异步流的方案！












