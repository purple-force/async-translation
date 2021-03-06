# 你不知道JS：异步

# 第二章：回调（Callbacks）

在第一章中，我们探究了JS中异步编程方面的一些术语和概念。我们的重点在于理解这一概念，即单线程（一次一个）事件轮询队列驱动着所有的”事件“（异步函数调用）。我们也探究了用并发模式来解释同时运行的事件链之间或者”进程“（任务，函数调用等）之间关系（如果有的话）的各种方式。

第一章中的所有例子采用的函数都是单独的、不可分割的操作单元，从而在函数内部，语句都是按照可预测的顺序运行的（高于编译器级）,但在函数顺序级，事件（即异步函数调用）可能以各种顺序运行。

在所有这些情形当中，函数充当了”回调“的角色，因为当处理队列中的某项任务时，它用于将事件轮询”回调入“程序当中。

毫无疑问，你已经注意到了，回调是迄今为止JS中表示和管理异步最普遍的方式了。诚然，回调是该语言中最重要的异步模式。

无数的JS程序，甚至是非常复杂的程序，它们的异步都是建立在回调基础上的，别无其它（当然也包括我们在第一章中所探究的并发交互模式）。回调函数是JS异步的驮马（译者注：意指绝大多数的异步是通过回调实现的），并且它做的还不错。

除了...回调函数也并不是没有缺点。很多开发者对更好的异步模式promise（双关，译者注：既指承诺，又指ES6中引入的Promise异步）感到十分激动。
但如果你不理解它抽象的是什么，以及为什么这样，就不可能有效地使用任何抽象。（译者注：好抽象！！！）

本章中，我们会深入探究下这些东西，以促使我们研究为什么更复杂的异步模式很有必要以及更受期待。

## 延续（Continuations）

让我们回到第一章中开始时的异步回调例子，但为了说明一点，让我稍微修改一下：

```javascript
// A
ajax( "..", function(..){
    // C
} );
// B
```

`// A`和`// B`代表程序的第一部分（即*现在*），`// C`代表程序中的第二部分（即*以后*）。第一个部分立即执行，然后有一个不确定时间的”暂停“。在未来的某个时候，如果Ajax调用完成，程序会从停下的地方开始，继续第二部分。

换句话说，回调函数封装了程序的*延续*。

让我们再简化下代码：

```javascript
// A
setTimeout( function(){
    // C
}, 1000 );
// B
```

等一下，问问自己该如何（向其他对JS如何运作知之甚少的人）描述程序的运行方式。继续，大声点。这是一次很好的尝试，会让我接下来的观点更有意义。

绝大多数读者目前的想法也只能达到这种程度：”执行A，然后设置一个1000毫秒的定时器，之后一旦触发，执行C“。你的解释有多接近？

你可能发现自己又修改成：”执行A，设置个1000毫秒的定时器，之后执行B，然后待定时器触发，执行C“。这比第一版更精确，你能注意到其中的差异吗?

即使第二版更精确，这两个版本都不足够以想法吻合代码、代码吻合JS引擎的方式解释这段代码。这种断层既微妙又意义重大，是理解表达和管理异步的回调缺点的关键。

一旦我们以回调函数的形式引入一个延续（Continuations）（或者如许多程序那样引入好几个），就会在脑中所想的和代码运行的方式之间产生分歧。任何时候，这两个的分歧都不可避免地让我们的代码更难以理解，推论，调试和维护。

## 序列化的大脑（Sequential Brain）

我很确定绝大多数读者曾经听过某些人说（甚至你也说过），”我是个一心多用的人“（译者注：指能同时处理多个任务）。从幽默的（比如，愚蠢的边拍脑袋边揉肚子的孩子游戏）到平常的（边走边嚼口香糖），再到非常危险的（边开车边发短信），形如这些种种。

但我们是一心多用的人吗？我们真的可以同时做并思考两件有意识的事情吗？我们大脑最高等级的功能中有并行多线程在运行吗？

答案可能让你感到惊讶：**或许不是**。

我们的大脑并不是那样设定的。尽管许多人（尤其是A类型人格的人）不愿承认，我们是一心一用的人。我们在任一时刻只能想一件事情。

我不是要讨论所有无意识的、潜意识的、自动的大脑功能，比如心跳、呼吸和眨眼睛等。这些都是维持生命所必须的重要任务，但我们不会特意分配脑力到这些事情上。谢天谢地，就算我们在三分钟内查看了15次社交网络提示，我们的大脑也会在幕后（多线程）处理所有这些重要的任务。

我们关注的反而是此刻心里幕前（译者注：相比于幕后）在处理什么任务。对我而言，我正在写这本书。此时此刻我还做着其它更高级的脑部功能吗？不，并没有。我很快并且很容易分心--最近几段文字已经分心好几次了！

当我们假装多任务时，比如在和朋友或者家人打电话时打字，我们所做的其实是在尽可能快地在多个情境中来回切换。换句话说，我们在两个或更多的任务之间持续的快速切换，处理很短很小的任务块。这一过程如此之快，以至于从表面看来，我们好像在并行地做这些事情。

对你而言，听起来是不是怀疑有点像异步事件并发（就跟JS中发生的差不多）？如果不是，回去重读第一章！

事实上，将复杂的神经世界简化为我想在此讨论的东西，一种简单的方式是，我们的大脑就像是事件轮询队列。

如果你把我写的每个字母（或者单词）想象是一个异步事件，对我而言，就在这一句话里，我的大脑思维就有可能多次被其它事件打断，比如我自己的感觉，亦或者其它随机的想法。

但每次我都不会被打断并拉到另一个”进程“当中去（谢天谢地--要不然就不会写这本书了）。但我经常感觉到我的大脑在各种不同的情境（即”进程“）间不断切换。如果JS引擎有感觉的话，它也会感到很痛苦。

### 执行 VS 计划（Doing Versus Planning）

好，我们可以将大脑想像为以单线程事件轮询队列的方式运行的，就像JS引擎一样。听起来挺搭的。

但我们需要更细致地分析其中的微妙差异。在如何规划各种任务和如何执行任务之间，有个巨大的、可见的差异。

再拿我写文章这一比喻来说，我粗略的罗列了下计划，按照心中想到的点按顺序一直写一直写。在写作过程中，我并没有对遇到的任何中断或者非线性活动（译者注：指跟写作无关的东西）作计划。然而，我的大脑一直在切换。

即使从操作层面来说，我的大脑是异步事件的，我们似乎喜欢以同步、序列化的方式规划任务。”我要去商店，然后买一些牛奶，然后把衣物交给干洗店“。

你可能注意到，这种高层级的想法（计划）从形式上来看并不是异步事件。实际上，对我们而言，单独按照事件来想很少见。反而，我们序列化地规划事情（A然后B然后C），就好像有某种时间阻塞强制B等A，C等B一样。

当开发人员写代码时，他们规划了一系列将要发生的事情。如果他们是优秀的开发人员，他们会仔细规划。”我要把`z`设为`x`的值，然后把`x`设为`y`的值“等等。

当我们一句一句地写出同步代码时，大概是这个样子：

```javascript
// swap `x` and `y` (via temp variable `z`)
z = x;
x = y;
y = z;
```

这三个赋值语句时同步的，因此`x=y`等`z=x`结束，而`y=z`等`x=y`结束。换句话说，这三个语句应该按照一个特定顺序执行，一个接着一个。万幸的是，这儿，我们不需要面对恼人的异步事件细节。如果需要的话，代码立刻会变得很复杂。

因此，如果同步的思维方式能够很好的对应到同步的代码语句，那么我们的大脑在规划异步代码时能做得有多好呢？


事实证明，我们代码表达异步（用回调）的方式并不能很好的符合我们大脑的同步规划事情这一行为。

你能像这样按时间线实际想象下计划的待做事件吗？

> "我要去趟商店，但路上我会打个电话，那么’嗨，妈妈‘，在她说话的时候，我会在GPS上查询商店的地址，但是会花几分钟加载，之后我调低收音机的音量，以便于我能够听清楚妈妈说话，然后我意识到我忘了穿夹克，外面挺冷的，但没关系，继续开，并且和妈妈打电话。之后听到叮的一声，提醒我系紧安全带，’是的，妈妈，我正在系安全带，我一直都系的‘，啊哈，最终GPS找到的正确的方向，现在..."

尽管在日常生活中，以这种方式规划、思考做什么以及做的顺序，听起来有点可笑，然而这正是我们大脑运行的方式。记住，这不是多任务，仅仅是快速切换。

对于开发人员来说，写异步事件代码，尤其是都是采用回调实现的话，是很困难的。究其原因，是因为意识流式的思考/计划不符合我们绝大多数人的本性。

我们是按照一步一步的顺序来想事情的，一旦我们从同步转向异步的时候，代码中可用的工具（回调）不是按照一步一步的方式来表达的。

这就是为什么很难用回调准确地编写和解释异步代码：因为不是我们大脑思考问题的方式。

**注意：** 

唯一一件比不知道代码为什么异常更糟的事情是，不知道为什么刚开始的时候好好的！这是经典的”纸牌屋“心理：”它运行正常，但不知道为什么，因而也就没人去碰它了！“你可能听说过，”他人即是地狱“（萨特），编程人员稍微改动一下，”他人的代码就是地狱“。我真的认为：”不了解自己的代码才是地狱“。而回调则是罪魁祸首。

### 嵌套/链式回调（Nested/Chained Callbacks）

考虑如下代码：

```javascript
listen( "click", function handler(evt){
    setTimeout( function request(){
        ajax( "http://some.url.1", function response(text){
            if (text == "hello") {
                handler();
            }
            else if (text == "world") {
                request();
            }
        } );
    }, 500) ;
} );
```

像这样的代码你一定很熟悉，我们用三个函数嵌套在一起，每一个代表一个异步操作（任务，”进程“）。

这种代码常称为”回调地狱“（callback hell），有时也成为”末日金字塔“（pyramid of doom）（因为由于嵌套缩进，看起来像一边倾斜的三角形）。

但是”回调地狱“和嵌套/缩进几乎没什么关系。问题远不止如此。随着深入本章的其余部分，我们会明白其中的道理。

首先，我们等待”click“事件，之后等待定时器触发，之后再等待Ajax响应返回，此时可能从头再来一遍。

第一眼看上去，这段代码似乎很自然地将异步对应到序列化的大脑计划方式上去。

首先（*现在*）,我们：

```javascript
listen( "..", function handler(..){
    // ..
} );
```

*之后*，我们：

```javascript
setTimeout( function request(..){
    // ..
}, 500) ;
```

再*之后*，我们：

```javascript
ajax( "..", function response(..){
    // ..
} );
```

最终（再*之后*），我们：

```javascript
if ( .. ) {
    // ..
}
else ..
```

但以这种线性方式解释这段代码有些问题。

首先，很巧合，我们的示例代码是一步一步来的（1,2,3,4...）。在实际的异步JS编程中，通常有许多干扰项出现，使得我们在从一个函数跳到另一个函数的时候，需要在脑中灵巧地绕过这些干扰项。在这种回调负担下理解异步流不是不可能，但即使有过多次实践，也不会是一件很自然和容易的事。

另外，还有一些更深层次的问题，这在实例代码中并不明显。让我们举另一个例子（伪代码）来说明：

```javascript
doA( function(){
    doB();

    doC( function(){
        doD();
    } )

    doE();
} );

doF();
```

尽管有经验的人能够正确地识别运行的顺序，我打赌第一眼看上去的时候还是有点困惑的，可能在心里想会儿才行。代码将会按以下顺序运行：

+ `doA()`
+ `doF()`
+ `doB()`
+ `doC()`
+ `doE()`
+ `doD()`

你第一次看到代码时做对了吗？

好，有些人会想我的函数命名并不公平，故意把人引入迷途。我发誓我只是按照从上到下的方式按序命名的。那我们再试一次：

```javascript
doA( function(){
    doC();

    doD( function(){
        doF();
    } )

    doE();
} );

doB();
```

现在，我们已经按照实际执行的字母顺序重命名了。但我打赌即使对这个情形有经验了，对多数读者来说，通过跟踪`A -> B -> C -> D -> E -> F`的顺序并不自然。同样，你的眼睛在代码片段之间痛苦地跳来跳去，是吗？

但即使你认为看起来还算自然，仍然有个更危险的东西可能捣乱，你注意到是什么了吗？

假如`doA(..)`或`doD(..)`不是我们认为的异步呢？哦，现在顺序不同了。如果它们两个都是同步的（有时可能是，依程序执行的条件而定），现在顺序是`A -> C -> D -> F -> E -> B`。

似乎听到了成千上万的JS程序员一声轻微的叹息，他们刚刚以手掩面。

是嵌套的问题吗？是它让异步流追踪变得困难了吗？当然，部分是。

但让我们重写之前的嵌套事件/定时器/Ajax，不用嵌套：

```javascript
listen( "click", handler );

function handler() {
    setTimeout( request, 500 );
}

function request(){
    ajax( "http://some.url.1", response );
}

function response(text){
    if (text == "hello") {
        handler();
    }
    else if (text == "world") {
        request();
    }
}
```

相比于之前的嵌套/缩进，这种形式的代码并没有变得更容易识别，很容易受”地狱回调“的影响。为什么？

当序列化地解释这段代码时，我们不得不跳过一个又一个函数，围绕着基本代码来“看”序列流。记住，这只是最好情况下的简化代码。我们都知道真正的异步JS程序通常更复杂，使得解释这种规模代码的顺序变得更困难。

另一个要注意的是：为了将步骤2,3,4串联起来以便连续发生，回调函数给我们的唯一启示是将步骤2硬编码进步骤1，步骤3硬编码进步骤2，步骤4硬编码进步骤3，等等。硬编码并不一定是坏事，如果步骤2总是导致步骤3发生的话。

但硬编码肯定会使程序变得脆弱，因为它并不会对可能导致程序步骤运行偏差的错误负责。比如，如果步骤2失败了，则永远也不会运行到步骤3，更没有重试步骤2了，或者移到一个可选的错误处理流中，诸如此类。

所有这些问题你可以通过手动地编入到每一步当中去解决，但那样的代码通常非常重复，在程序的其它步骤或者异步流中难以复用。

尽管可能是以序列化的方式（这样，然后这样，然后这样）计划一系列任务，由于我们大脑事件化的特点，流程控制的恢复/重试/分叉变得毫不费力。如果你出去办事，发现把购物清单落在家了。一天并没有结束，因为你没有事先计划。你的大脑只会简单地绕过这件小事：回到家、拿上清单，然后立刻回到商店去。

但是手动硬编码的回调函数由于其脆弱的特性，很不优雅。一旦你指定（即预先计划）完所有可能的结果，这段代码就会变得很难维护和升级。

**那**才是“回调地狱”的真正含义！嵌套/缩进只是基本的障眼法。

那还不够，我们还没提到当两个或更多的回调链同时发生时会出现什么情况。或者第三步扩展到有“门”（gate）或“闩”（latch）的并行回调中，或者...OMG，我脑袋疼，你们呢？

此刻，你注意到了吗？我们序列化、阻塞性的大脑计划行为并不能很好地对应到回调导向的异步代码。这是阐述回调时的第一个主要缺陷：代码中采用回调表示异步的方式，我们的大脑必须很努力才能跟上。

## 信任问题（Trust Issues）

序列化的大脑行为和回调驱动的异步代码之间的不吻合仅仅是回调函数的一个问题。还有更深层次的东西需要我们关注。

让我们再看看回调函数作为延续这一概念（即第二部分）：

```javascript
// A
ajax( "..", function(..){
    // C
} );
// B
```

在JS主程序的控制下，`// A`和`// B`*现在*执行，但是`// C`推迟到*以后*执行，而且是在第三方的控制之下--这里指`ajax(..)`函数。一般来说，这种交出控制权的方式不会出现太大问题。

尽管很少出问题，但不要被骗了，以为交出控制权不是大问题。实际上，这是以回调为驱动的设计最糟糕（也是最微妙）的问题之一。有时，`ajax(..)`（即你把回调交给的一方）并不是你写的或直接控制的函数。很多时候，是由第三方提供的utility。

当你把程序的部分执行控制权交给第三方时，我们称之为“控制反转”。在你的代码和第三方utility间存在着一种无以言表的“契约”--一些你想维护的东西。

### 五个回调的故事(Tale of Five Callbacks)

为什么这是个大问题，可能不是很明显。让我构建一个夸大的场景来说明缺乏信任的危害。

假如你是一名开发人员，负责为一个售卖高价电视机的网站开发一个电子结账系统。你已经把所有结账系统的页面都做好了。在最后一页，当用户点击“确认”购买电视机的时候，你需要调用第三方函数（假如由某个分析跟踪公司提供）来跟踪此次订单。

你注意到他们提供了一个看起来像异步的跟踪程序，或许为了考虑性能，你需要传入一个回调函数。在回调函数当中，你需要对用户的信用卡进行扣款并且展示感谢页面。

代码可能像这样：

```javascript
analytics.trackPurchase( purchaseData, function(){
    chargeCreditCard();
    displayThankyouPage();
} );
```

很容易，是不是？写完代码，测试正常，你把它部署到生产环境。万事大吉！

6个月过去了，没有问题。你差不多都忘了你写过那段代码。一天早上，工作之前，你在咖啡店悠悠哉哉地喝着拿铁。这时，一声电话铃响，你的老板让你扔掉咖啡，马上回去工作。

当你到时，你发现一个高级用户买的电视机被扣了五次款，并且他很心烦。客服已经道过歉并且退款了。但你的老板想知道这是怎么回事。“这段代码我们不是测试过了吗？”

你甚至不记得写过的代码。但是你回头仔细查看，到底哪出问题了。

在查看了一些日志后，你得出结论，唯一的解释就是utility不知什么原因调用了五次回调函数，而不是一次。他们的文档中从没提过这事。

你很沮丧，联系了他们的客服，他们当然和你一样吃惊。他们同意上报给他们的开发人员，并且许诺很快给你回复。第二天，你收到一个很长的邮件，邮件中解释了他们的发现，然后你把它呈递给你的老板。

很明显，分析公司的开发人员过去一直在某种情况下做一些代码实验，可能在超时之前，每秒钟重执行下回调函数，总共5秒钟。他们从没想过把它放到生产环境中，但不知什么原因，确实放上去了。他们感到很不好意思并道了歉。他们披露了大量关于如何定位问题的细节以及将要怎么做，确保不会再发生了。等等等等。

接下来呢？

你跟老板说了这件事，但他对这件事感到不是很爽。坚持要你不能再相信他们了（这也是你要说的），你也勉强同意，并且需要搞明白如何保护结账代码，以免再受其害。

做了些修改之后，你写了一些简单的专用代码，如下，团队成员对此也很乐意：

```javascript
var tracked = false;

analytics.trackPurchase( purchaseData, function(){
    if (!tracked) {
        tracked = true;
        chargeCreditCard();
        displayThankyouPage();
    }
} );
```

**注意：**看过第一章之后，你可能觉得很熟悉。因为我们创建了一个闩（latch）来处理突然多次并发执行回调函数的情况。

但一个QA工程师问了：“要是他们从来不调用回调函数呢？”哎呦。谁都没想过这个问题。

你开始考虑他们调用你的回调函数时可能出现的各种情况。以下是你想出来的utility可能出错的地方：

+ 太早调用回调函数（在跟踪之前）
+ 太晚调用（或者从不调用）
+ 调用太少或者太多次（就像你遇到的问题！）
+ 无法向回调函数传递任何必要的环境/参数
+ 掩盖可能发生的错误/异常（译者注：即有异常却不报出来）
+ ...

好麻烦的一长串列表，确实是这样。你可能慢慢意识到需要在每个单独的回调（传入到你不确定是否能相信的utility中去）当中写大量专门的代码，好痛苦。

现在你对“地狱回调”的理解更深一层了。

### 不只是别人的代码（Not Just Others' Code）

此时，可能有人会怀疑，我所说的这种情况是否是个大问题。或许你不怎么和第三方utility打交道。或许你使用的是版本化的API或者自托管的库，因此程序的行为不会被除你之外的人改变。

那么，仔细想想：你真的能相信你理论上控制的utility吗（在你自己的代码库中）？

这样想想：为了减少意外问题，在一定程度上，我们绝大多数都会在函数内部对输入参数作一些防御性检查。

过度信任输入值：

```javascript
function addNumbers(x,y) {
    // + is overloaded with coercion to also be
    // string concatenation, so this operation
    // isn't strictly safe depending on what's
    // passed in.
    return x + y;
}

addNumbers( 21, 21 );   // 42
addNumbers( 21, "21" ); // "2121"
```

防范不信任的输入：

```javascript
function addNumbers(x,y) {
    // ensure numerical input
    if (typeof x != "number" || typeof y != "number") {
        throw Error( "Bad parameters" );
    }

    // if we get here, + will safely do numeric addition
    return x + y;
}

addNumbers( 21, 21 );   // 42
addNumbers( 21, "21" ); // Error: "Bad parameters"
```

或许在安全的基础上更友好一点：

```javascript
function addNumbers(x,y) {
    // ensure numerical input
    x = Number( x );
    y = Number( y );

    // + will safely do numeric addition
    return x + y;
}

addNumbers( 21, 21 );   // 42
addNumbers( 21, "21" ); // 42
```

然而，当你着手这么做的时候，你会发现这些对函数输入的检查/规范化相当常见，甚至在理论上我们完全相信的代码中。从粗浅的意义上来说，这种编程等价于地缘政治学中的“信任但要验证”的原则。

那么，很显然，我们在包含异步函数回调的的程序中也该这么做，不仅仅是外部代码，还有我们通常认为“在我们控制之下”的代码，不是吗？**当然是的。**

但回调几乎没有提供任何东西来帮助我们。我们必须自己构建所有的检查机制，最终都是一些在每个回调中不断重复的样板/开销。

回调最令人头疼的问题是*控制权反转*，这导致了所有这些信任的完全崩溃。

如果你的代码中使用了回调，特别是有第三方utility的，如果对*控制权反转*信任问题没有作一些缓和逻辑控制，你的代码中就有bug了，即使它现在可能不出现。潜在的bug还是bug。

确实是地狱。

## 试图拯救回调（Trying to Save Callbacks）

有几种回调设计试图解决我们提到的部分（不是全部）信任问题。拯救回调免于自我崩溃是个勇敢但注定失败的尝试。

例如，关于更优雅的错误处理，一些API设计提供了分离式回调（一个是成功回调，一个是错误回调）：

```javascript
function success(data) {
    console.log( data );
}

function failure(err) {
    console.error( err );
}

ajax( "http://some.url.1", success, failure );
```

采用这种方式设计的API中，通常`failure()`是可选的，如果没有提供，则假定你想掩盖这个错误。呃。

**注意：** ES6 Promise API采用这种分离式回调的设计。在下一章我们会详细讨论ES6 Promises。

另一种常用的回调模式称为“错误优先类型”（有时叫做“Node类型”，因为这是几乎所有Node.js API的约定形式），其中，回调函数的第一个参数保留用作错误对象（如果有的话）。如果成功，这个参数就为空/falsy(译者注：指能转化为false)（之后的参数都是成功数据），如果有错误，则设为第一个参数/truthy（译者注：指能转化为true）（通常只有该错误对象没有其它参数传入）：

```javascript
function response(err,data) {
    // error?
    if (err) {
        console.error( err );
    }
    // otherwise, assume success
    else {
        console.log( data );
    }
}

ajax( "http://some.url.1", response );
```

以上两种情况，有几点需要注意：

首先，并没有真正解决可能出现的大多数信任问题。对回调函数不需要的重复调用操作既没有阻止，也没有过滤。另外，情况变得更糟了，你可能同时获得成功和失败信号，或者一个都得不到，你仍然需要针对这些情况写一些额外代码。

另外，一个不可忽略的事实是，尽管这是一个你可以采用的标准模式，但它肯定更冗长并且样板化，没法进行太多的复用。因此，在应用程序的每个回调中，你需要不厌其烦地编写那些代码。

那从不调用这一信任问题呢？如果这是个问题（并且应该是个问题！），你可能需要设置一个定时器来取消事件。你可以采用某个utility（仅作概念展示）来帮助你处理这个问题：

```javascript
function timeoutify(fn,delay) {
    var intv = setTimeout( function(){
            intv = null;
            fn( new Error( "Timeout!" ) );
        }, delay )
    ;

    return function() {
        // timeout hasn't happened yet?
        if (intv) {
            clearTimeout( intv );
            fn.apply( this, [ null ].concat( [].slice.call( arguments ) ) );
        }
    };
}
```

以下是如何使用：

```javascript
// using "error-first style" callback design
function foo(err,data) {
    if (err) {
        console.error( err );
    }
    else {
        console.log( data );
    }
}

ajax( "http://some.url.1", timeoutify( foo, 500 ) );
```

另一个信任问题称为“太早调用”。在具体应用方面，可能会在某些重要任务完成之前就被调用。但更通常来说，这个问题等价于utility中*现在*（同步）或者*以后*（异步）运行你提供的回调函数。

这种同步或异步不定的行为总是使得bug难以追踪。在某些圈子中，采用科幻世界中的精神诱导怪兽Zalgo来描述同步/异步噩梦。“不要释放Zalgo！”是很常见的诉求，听起来好像是建议：总是异步调用回调函数，即使是在下一个事件轮询中“立即”执行的，这样所有的回调函数都是异步的。

**注意：**若想获取更多关于Zalgo的信息，请看Oren Golan的“Don't Release Zalgo!”（[https://github.com/oren/oren.github.io/blob/master/posts/zalgo.md](https://github.com/oren/oren.github.io/blob/master/posts/zalgo.md)）（译者注：原文好像已经不存在了）和 Isaac Z. Schlueter的“异步接口设计”（[http://blog.izs.me/post/59142742143/designing-apis-for-asynchrony](http://blog.izs.me/post/59142742143/designing-apis-for-asynchrony)）。

考虑如下：

```javascript
function result(data) {
    console.log( a );
}

var a = 0;

ajax( "..pre-cached-url..", result );
a++;
```

这段代码会输出`0`(同步回调调用)还是`1`(异步回调调用)？根据...情况。

从这你就可以看到Zalgo的不确定性对任一JS程序造成的威胁。因此，“不要释放Zalgo”听起来有点愚蠢，但是这是一个非常常见且可靠的建议。总是采用异步。

要是你不知道所涉及的API是否是异步执行的呢？你可以写个utility如`asyncify(..)`,仅作概念展示：

```javascript
function asyncify(fn) {
    var orig_fn = fn,
        intv = setTimeout( function(){
            intv = null;
            if (fn) fn();
        }, 0 )
    ;

    fn = null;

    return function() {
        // firing too quickly, before `intv` timer has fired to
        // indicate async turn has passed?
        if (intv) {
            fn = orig_fn.bind.apply(
                orig_fn,
                // add the wrapper's `this` to the `bind(..)`
                // call parameters, as well as currying any
                // passed in parameters
                [this].concat( [].slice.call( arguments ) )
            );
        }
        // already async
        else {
            // invoke original function
            orig_fn.apply( this, arguments );
        }
    };
}
```

可以这样使用`asyncify(..)`：

```javascript
function result(data) {
    console.log( a );
}

var a = 0;

ajax( "..pre-cached-url..", asyncify( result ) );
a++;
```

不管Ajax请求是在缓存中，进而试图立即调用回调函数，还是必须从远端获取之后异步执行，这段代码总是输出`1`而不是`0`--`result(..)`不得不异步执行，这就意味着`a++`有机会在`result(..)`之前运行。

耶，另一个信任问题“解决了”！但很低效，更多的样板代码使得你的项目臃肿不堪。

这就是一遍一遍采用回调函数的故事。回调可以很好地完成你的任何所需，但需要花很大功夫，并且这份努力通常比你应该花费的要多得多。

你可能希望有内建的API或者其它语言机制来处理这个问题。最终，ES6来到幕前，给出了解决方案，那么继续往下读！

## 回顾（Review）

回调是JS中重要的异步编程单元。但随着JS的日趋成熟，回调函数并不足以支撑异步编程的演进。

首先。我们的大脑是以序列化、阻塞性、单线程（从语义上来说）的方式计划事情的，但是回调函数是以非线性、非序列化的方式表达异步流的，这使得推演代码更困难。会产生糟糕bug的代码不利于推演代码。

我们需要以一种更同步、序列化、阻塞性的方式来表达异步，就像我们大脑所做的那样。

其次，更重要的一点是，回调深受*控制权反转*之害，因为控制权会隐式地转给其它方（通常是一个不受你控制的第三方utility！）来继续运行程序的*延续*。这种控制权反转给我们造成了一系列信任问题，比如回调函数的调用次数是否超过了我们的预期。

可以通过写些专门的代码来解决信任问题，但它本不应该这么困难，同时，也使得代码更加臃肿且难于维护。直到你碰到bug时你才觉得防护工作做得不够。

对于**所有这些问题**，我们需要一个通用的解决方案,一个一旦创建就可以被多个回调复用，不需要额外样板开销的解决方案。

我们需要一些比回调更好的东西。迄今为止，回调表现得还行，但JS的未来需要更复杂和更强大的异步模式。本书的下一章将会讨论那些新出现的演进。





















