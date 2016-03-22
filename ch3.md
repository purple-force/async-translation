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

此外，一旦Promise解析了，其状态永远不变--自那一时刻起，变成了*不可更改的值*--可以按需多次观测。

**注意：** 因为Promise一旦解析，就无法从外面更改其状态了，这样把值传递给第三方就很安全，因为无法无意或恶意地修改它。尤其是当有多个当事方观测这一个Promise的解析结果时更是如此。一方无法影响另一方对Promise解析的观测。不可变性听起来似乎像一个学术课题，但它其实是Promise设计中十分重要的方面之一，不能随意忽略。

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

**注意：** 如果你之前已经看过基于Promise的代码，你可能认为最后两行代码可以采用链式方式写作`p.then( .. ).then( .. )`，而不是`p.then(..); p.then(..)`。



