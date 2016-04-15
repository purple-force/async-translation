# 你不知道JS：异步

# 第四章：生成器（Generators）

在第二章，我们明确了采用回调表示异步流的两个关键缺点：

+ 基于回调的异步和我们大脑按步规划任务的方式不相吻合。
+ 由于*控制权反转*，回调不可信且无法组合。

在第三章中，我们详细描述了Promise如何反逆转回调的*控制权反转*问题的，是我们重新获得了可信任/可组合的能力。

现在，我们把目光转向一种序列化的、看起来像同步的异步流表示形式。实现这一功能的“魔法”是ES6 **生成器**。

## 打破运行直至结束（Breaking Run-to-Completion）

在第一章，我们解释了一个JS开发者在代码中普遍依赖的期望：一旦函数开始执行，它会一直运行到结束，在此之间没有其它代码（译者注：不包括其中的错误代码哈）能够中断和执行。

尽管看起来有点奇怪，ES6引入了一个新的函数类型，它的行为不同于Run-to-Completion。这种新类型的函数称为“生成器”。

为理解其含义，考虑下这个例子：

```javascript
var x = 1;

function foo() {
    x++;
    bar();              // <-- what about this line?
    console.log( "x:", x );
}

function bar() {
    x++;
}

foo();                  // x: 3
```

在这个例子中，我们很确定`bar()`会在`x++`和`console.log(x)`之间运行。但要是`bar()`不在那呢？很显然，结果是`2`，而不是`3`。

现在，动动你的脑筋。要是`bar()`语句不存在，但是由于某种原因，仍然在`x++`和`console.log(x)`语句之间运行？怎么可能？

在**抢占式**多线程语言中，`bar()`中断这两个语句（译者注：指`x++`和`console.log(x)`语句）并且就在两者之间执行是可能的。但JS不是抢占式的，也不是（目前）多线程的。另外，如果`foo()`本身在代码的某个部分可以“暂停”，则这种“中断”（并发）的**协作**（cooperative）形式是可能的。

**注意：** 我使用了“协作”（cooperative）一词，不仅仅是因为和传统的并发术语的联系（见第一章），还因为ES6语法中采用`yield`指明代码中的暂停点--表示一个礼貌性地*协作式*控制让步。

以下是实现此类协作并发的ES6代码：

```javascript
var x = 1;

function *foo() {
    x++;
    yield; // pause!
    console.log( "x:", x );
}

function bar() {
    x++;
}
```

**注意：**你可能看到绝大多数其它JS文档/代码会以`function* foo() { .. }`的形式声明一个生成器，而不是我此处使用的`function *foo() { .. }`--唯一的不同是`*`的文体位置。这两种格式从功能/语法上来说是一致的，和第三种形式，`function*foo() { .. }`（没有空格）也是一样的。这两种形式都有争议，但是我偏向于`function *foo..`，因为这与我以`*foo()`形式引用生成器的方式相吻合。如果我只说`foo()`，你不知道我是在说生成器还是一个普通的函数。完全是文体形式上的偏爱而已。

现在，我们该如何运行前面的代码，使得`bar()`运行到`*foo()`中的`yield`点？

```javascript
// construct an iterator `it` to control the generator
var it = foo();

// start `foo()` here!
it.next();
x;                      // 2
bar();
x;                      // 3
it.next();              // x: 3
```

好，这两段代码中有些新的并且有些令人困惑的东西，因此我们需要了解的更多才行。但在我们解释ES6生成器的不同机制/语法之前，让我们简单过下代码的行为流程：

1. `it = foo()`操作还*没有*执行`*foo()`生成器（译者注：指内部的代码），而是仅仅构建了控制执行的*迭代器（iterator）*。关于*迭代器*的更多细节会在后面提及。
2. 第一个`it.next()`启动了`*foo()`生成器，执行了`*foo()`中第一行的`x++`。
3. `*foo()`在`yield`语句处暂停，即第一个`it.next()`调用结束的点。此时`*foo()`仍然在运行并活动，只是处在暂停状态。
4. 我们检查了`x`的值，现在是`2`。
5. 我们调用`bar()`，通过`x++`又增加了`x`的值。
6. 我们再次检查`x`的值，现在是`3`。
7. 最后的`it.next()`调用从暂停的地方恢复了`*foo()`生成器，运行`console.log(..)`语句，使用当前`x`的值`3`。

很明显，`*foo()`启动时，并*没有*运行直至结束（run-to-completion）--它在`yield`处暂停。我们随后恢复了`*foo()`，让它结束，但那并不是必须的。

因此，生成器是一种特殊类型的函数，可以启动和停止一次或多次，甚至没必要结束。尽管它为什么如此强大还不是很明显，随着我们深入本章的其它部分，我们会发现，它会是用来构建生成器异步流控制（generators-as-async-flow-control）模式的重要构建块之一。

### 输入和输出（Input and Output）

生成器函数是一种特殊的函数，我们刚刚简单提及了它的新的处理模式。但是它仍然是函数，这意味着它仍然有一些不变的基本原则--即，仍然接收参数（即“输入”），并且仍会返回一个值（即“输出”）：

```javascript
function *foo(x,y) {
    return x * y;
}

var it = foo( 6, 7 );

var res = it.next();

res.value;      // 42
```

我们向`*foo()`中传入了实参`6`和`7`，分别对应于参数`x`和`y`。`*foo()`向调用代码返回值`42`。

现在，我们看一下，相比于普通函数，生成器的激活方式有何不同。`foo(6,7)`看起来很熟悉。但有些微妙区别，不像普通函数，`*foo(..)`还没真的运行。

我们只是创建了一个*迭代器（iterator）*对象，将它赋给变量`it`，用来控制`*foo(..)`生成器。之后我们调用`it.next()`，命令`*foo(..)`生成器从当前位置前进到下一个`yield`或者生成器末尾后停止。

`next(..)`调用结果是个包含`value`属性的对象，存放着`*foo(..)`返回的任何值（如果有的话）。换句话说，`yield`会在程序执行中间将值发送至生成器之外，有点像中间的`return`。

再说一次，为什么我们需要这种完全不直接的*迭代器（iterator）*对象来控制生成器呢，原因还不是很明显。我们会搞清楚的，我保证。

#### 迭代信息传递（Iteration Messaging） 

除了接收实参和返回值，生成器内建有更强大的输入/输出信息传递能力，通过`yield`和`next(..)`。

考虑如下代码：

```javascript
function *foo(x) {
    var y = x * (yield);
    return y;
}

var it = foo( 6 );

// start `foo(..)`
it.next();

var res = it.next( 7 );

res.value;      // 42
```

首先，我们向`x`传入`6`。之后我们调用`it.next()`，启动`*foo(..)`。

在`*foo(..)`内部，`var y = x ..`语句开始执行，但是之后碰到了`yield`表达式。那时，`var y = x ..`语句暂停了`*foo(..)`（在赋值语句中间！），请求调用代码为`yield`表达式提供一个结果值。之后，我们调用`it.next( 7 )`，将`7`传回作为暂停的`yield`的结果。

那么，此时，赋值语句的本质上就是`var y = 6 * 7`。现在，`return y`返回值`42`作为`it.next( 7 )`调用的结果。

即使对有经验的JS开发者来说，有些非常重要但很容易混淆的东西需要注意：从你角度看，`yield`和`next(..)`调用二者不相匹配。通常，`next(..)`调用比`yield`语句多一个--前面的代码中有一个`yield`和两个`next(..)`调用。

为什么会不匹配？

因为第一个`next(..)`总是用来启动生成器，运行到第一个`yield`。但是执行第一个暂停的`yield`表达式的是第二个`next(..)`调用，执行第二个暂停的`yield`表达式的是第三个`next(..)`调用，以此类推。

##### 两个问题的故事（Tale of Two Questions）

事实上，主要关注的代码会影响你对是否有感观上不匹配的判断。

只考虑生成器代码：

```javascript
var y = x * (yield);
return y;
```

这里**第一个**`yield`简单地*问一个问题*：“我应该在这里插入哪个值？”

谁来回答这个问题呢？好吧，**第一个**`next()`已经让生成器运行到这里了，因此，很明显无法回答这个问题。因此**第二个**`next()`调用必须回答由**第一个**`yield`提出的问题。

看到不匹配了吗--第二个对第一个？

让我们变换一下角度。不要从生成器角度看，而从迭代器的角度看。

为了恰当地说明这个角度，我们也需要解释下信息可以向两个方向传递--作为表达式，`yield..`可以响应`next(..)`调用，向外发出信息，`next(..)`也可以将值发送给暂停的`yield`表达式。考虑如下代码，稍微作了修改：

```javascript
function *foo(x) {
    var y = x * (yield "Hello");    // <-- yield a value!
    return y;
}

var it = foo( 6 );

var res = it.next();    // first `next()`, don't pass anything
res.value;              // "Hello"

res = it.next( 7 );     // pass `7` to waiting `yield`
res.value;              // 42
```

**在生成器执行期间**，`yield..`和`next(..)`对一起，充当了两路信息传递系统。

那么，只看*迭代器*代码：

```javascript
var res = it.next();    // first `next()`, don't pass anything
res.value;              // "Hello"

res = it.next( 7 );     // pass `7` to waiting `yield`
res.value;              // 42
```

**注意：** 我们没有向第一个`next()`调用传递值，那是有目的的。只有暂停的`yield`才能接收`next(..)`传递的值，当我们调用第一个`next()`时，**没有暂停的**`yield`来接收值。规范和所有兼容的浏览器只是静默的**丢弃**任何传入第一个`next()`的值。

第一个`next()`调用（没有传值给它）只是简单地*问了个问题*：“`*foo(..)`生成器*下一个*该给我的值是什么？”谁来回答这个问题呢？第一个`yield "hello"`表达式。

看到了吗？没有不匹配。

`yield`和`next(..)`调用之间有没有不匹配，取决于你认为*谁*来回答这个问题。

但等等！相比于`yield`语句，仍然多一个`next()`。因此最后一个`it.next(7)`调用再次发问，生成器产生的下一个值是什么。但是没有剩余的`yield`语句来回答了，不是吗？那么谁来回答呢？

`return`语句回答这个问题！

要是生成器中**没有**`return`--`return`不再像普通函数那样显得十分必要了--总有个假定的/隐式地`return;`（即`return undefined;`），充当了默认回答最后`it.next(7)`调用提出的问题的角色。

这些问题和回答--采用`yield`和`next(..)`两路信息传递--相当强大，但是这些机制如何和异步流控制连接在一起，还不是很清楚。我们会明白的！

### 多个迭代器（Multiple Iterators）

可能出现一种语法使用情况，即当你使用一个*迭代器*控制生成器时，你要控制的是声明的生成器函数本身。但很容易忽略些细微之处：每次构建一个*迭代器*，你都隐式地构建了一个生成器实例，由那个*迭代器*控制。

你可以同时让同一个生成器的多个实例同时运行，它们之间甚至可以交互：

```javascript
function *foo() {
    var x = yield 2;
    z++;
    var y = yield (x * z);
    console.log( x, y, z );
}

var z = 1;

var it1 = foo();
var it2 = foo();

var val1 = it1.next().value;            // 2 <-- yield 2
var val2 = it2.next().value;            // 2 <-- yield 2

val1 = it1.next( val2 * 10 ).value;     // 40  <-- x:20,  z:2
val2 = it2.next( val1 * 5 ).value;      // 600 <-- x:200, z:3

it1.next( val2 / 2 );                   // y:300
                                        // 20 300 3
it2.next( val1 / 4 );                   // y:10
                                        // 200 10 3
```

**警告：** 同一个生成器的多个实例并发运行，这种方式最常见的用法不是这样交互的，而是生成器生成自己的值，或许来自某些不相关的独立资源，不需要输入。我们会在下一节讨论更多关于值生成的内容。

让我们简单过下流程：

1. `*foo()`的两个实例同时启动，并且两个`next()`调用分别从`yield 2`语句中得到为`2`的`value`。
2. `val2 * 10`是`2 * 10`，被传入第一个生成器实例`it1`，因此`x`获得值`20`。`z`从`1`增加到`2`，之后`20*2`被`yield`出来，将`val1`设为`40`。
3. `val1 * 5`是`40 * 5`，被传入第二个生成器实例`it2`，因此`x`获得值`200`。`z`从`2`增加到`3`，之后`200*3`被`yield`出来，将`val2`设为`600`。
4. `val2 / 2`是`600 / 2`，被传入第一个生成器实例`it1`，因此`y`获得值`300`，之后打印出`20 300 3`，分别对应于`x y z`。
5. `val1 / 4`是`40 / 4`，被传入第二个生成器实例`it2`，因此`y`获得值`10`，之后打印出`20 10 3`，分别对应于`x y z`。

这是在你心里运行的一个“有趣”的例子。你弄清楚了吗？

#### 交叉（Interleaving）

回想下第一章中“Run-to-completion”一节的场景：

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
```

对于普通JS函数，当然`foo()`可以先运行完，`bar()`也可以先运行完，但是`foo()`不能将内部的语句穿插到`bar()`中。因此，之前的程序只能有两个可能结果。

然而，对于生成器，很明显，交叉是可能的（甚至在语句中）：

```javascript
var a = 1;
var b = 2;

function *foo() {
    a++;
    yield;
    b = b * a;
    a = (yield b) + 3;
}

function *bar() {
    b--;
    yield;
    a = (yield 8) + b;
    b = a * (yield 2);
}
```

根据控制`*foo()`和`*bar()`的*迭代器*调用的相对顺序不同，这个程序可能生成多个结果。换句话说，通过共享同一变量交叉两个生成器，实际上我们可以实现（以一种虚假的方式）第一章中理论上的“线程竞态”情形。

首先，我们实现一个辅助函数`step(..)`，用来控制*迭代器*。

```javascript
function step(gen) {
    var it = gen();
    var last;

    return function() {
        // whatever is `yield`ed out, just
        // send it right back in the next time!
        last = it.next( last ).value;
    };
}
```
`step(..)`初始化生成器，生成自己的`it`*迭代器*，之后返回了一个函数，该函数单步推进`迭代器`。另外，之前`yield`出的值被传回*下一步*中。因此，`yield 8`只会变成`8`，`yield b`只会变成`b`（`yield`的是什么就是什么）。

现在，只是为了好玩，让我们做个试验，来看看交叉`*foo()`和`*bar()`的不同代码块会有什么效果。我们先以基本的用例开始，确保在`*bar()`之前，`*foo()`已经完全运行完了（就像我们在第一章做的那样）：

```javascript
// make sure to reset `a` and `b`
a = 1;
b = 2;

var s1 = step( foo );
var s2 = step( bar );

// run `*foo()` completely first
s1();
s1();
s1();

// now run `*bar()`
s2();
s2();
s2();
s2();

console.log( a, b );    // 11 22
```

最终结果为`11`和`22`，和第一章中的版本一样。现在打乱下交叉顺序，看会如何改变`a`和`b`的最终值：

```javascript
// make sure to reset `a` and `b`
a = 1;
b = 2;

var s1 = step( foo );
var s2 = step( bar );

s2();       // b--;
s2();       // yield 8
s1();       // a++;
s2();       // a = 8 + b;
            // yield 2
s1();       // b = b * a;
            // yield b
s1();       // a = b + 3;
s2();       // b = a * 2;
```

在我告诉你结果之前，你能明白程序执行完后`a`和`b`的值是多少吗？不许作弊！

```javascript
console.log( a, b );    // 12 18
```
（译者注：可能有的读者认为结果是`12 24`，注意在第三次`s2()`调用完成后，`a`的值为`9`。此时`*bar()`中的暂停点在`b = a * (yield 2)`处，其已经变成了`b = 9 * (yield 2)`。若将`b = a * (yield 2)`改为`b = (yield 2) * a`，则最终结果为`12 24`。）
**注意：** 作为读者练习，重排`s1()`和`s2()`的调用顺序，看能有多少种其它结果。别忘了总是需要三次`s1()`调用和四次`s2()`调用。要想知道原因，回想一下之前讨论的`next()`和`yield`匹配问题。

很确定的是，你基本上不会故意创建这种级别的交叉混淆，因为代码太难理解了。但是这一练习很有趣，并且有助于对多个生成器共享作用域时并发运行方式的理解，因为有时候这种能力很有用。

我们会在本章末尾讨论更多关于生成器并发的细节。

## Generator'ing Values

在前一节，我们提到了生成器的一个很有意思的用法，用来生成值。这*不是*本章的重点，但如果我们不讲这些基本的，就是我们失职，因为这种用法契合其名称的本来意义：生成器。

我们稍微转向*迭代器*主题，但之后我们会绕回来讲讲它们是如何和生成器挂上钩的，以及使用生成器*生成*值。

### 发生器和迭代器（Producers and Iterators）

假设你要生成一系列值，每个值和前一个值有明确的关系。为了实现这个，你需要一个带状态的发生器（producer）来记住它给出的上一个值。

你可以直接采用类似函数闭包的方式来实现：

```javascript
var gimmeSomething = (function(){
    var nextVal;

    return function(){
        if (nextVal === undefined) {
            nextVal = 1;
        }
        else {
            nextVal = (3 * nextVal) + 6;
        }

        return nextVal;
    };
})();

gimmeSomething();       // 1
gimmeSomething();       // 9
gimmeSomething();       // 33
gimmeSomething();       // 105
```

**注意：** 此处`nextVal`的计算逻辑本应该很简单，但从概念上，我们想直到*下一个（next）*`gimmeSomething()`调用发生的时候再计算*下一个（next）* 值，因为通常来说，对于更持久的或资源受限值（而不是简单的`number`）发生器而言，这是一种资源泄露式的设计。

生成一个随机数字序列并不是太合实际的例子。但要是你想从数据源生成记录呢？代码基本上差不多。

事实上，这是一个很常见的设计模式，通常由迭代器解决。对于步进遍历一系列由发生器生成的值，*迭代器*是个定义良好的接口。JS中的迭代器接口，和其它绝大多数语言一样，每当你想要从发生器中取下一个值时，只需调用`next()`。

对于数字序列发生器，我们可以实现标准的*迭代器*接口：

```javascript
var something = (function(){
    var nextVal;

    return {
        // needed for `for..of` loops
        [Symbol.iterator]: function(){ return this; },

        // standard iterator interface method
        next: function(){
            if (nextVal === undefined) {
                nextVal = 1;
            }
            else {
                nextVal = (3 * nextVal) + 6;
            }

            return { done:false, value:nextVal };
        }
    };
})();

something.next().value;     // 1
something.next().value;     // 9
something.next().value;     // 33
something.next().value;     // 105
```

**注意：** 我们会在"Iterables"一节中解释为什么这段代码中需要`[Symbol.iterator]: ..`部分。从语法上来说，此处有两个ES6特性。首先，`[..]`语法称为*计算属性名（computed property name）*。它是一种对象字面量的定义方式，指定一个表达式，并用表达式的结果作为属性名。其次，`Symbol.iterator`是ES6预定义的特殊`Symbol`值之一。

`next()`调用返回一个包含两个属性的对象：`done`是个`boolean`值，表示*迭代器*的完成状态；`value`保存着迭代的值。

ES6还加了`for..of`循环，这意味着可以通过原生的循环语法来自动处理标准的的*迭代器*:

```javascript
for (var v of something) {
    console.log( v );

    // don't let the loop run forever!
    if (v > 500) {
        break;
    }
}
// 1 9 33 105 321 969
```

**注意：** 因为`something`*迭代器*总是返回`done:false`，这个`for..of`循环会永远运行，这就是我们放置一个`break`条件判断的原因。迭代器永不结束也没关系，但也有一些其它情况，比如*迭代器*会遍历一组有限数据集并且最终返回`done:true`。

每次迭代，`for..of`循环自动调用`next()`--它并不会向`next()`中传递任何值--并且依据接收的`done:true`自动终止迭代。对于遍历数据集非常方便。

当然，你也可以手动遍历迭代器，调用`next()`并检查`done:true`条件来判断何时停止：

```javascript
for (
    var ret;
    (ret = something.next()) && !ret.done;
) {
    console.log( ret.value );

    // don't let the loop run forever!
    if (ret.value > 500) {
        break;
    }
}
// 1 9 33 105 321 969
```

**注意：** 这种手动的`for`方法当然比`for..of`遍历难看，但是其优点是，如果需要，你可以向`next(..)`调用中传递值。

除了实现自己的*迭代器*，JS（自ES6起）中许多内建的数据结构，比如`array`，同样有默认的*迭代器*：

```javascript
var a = [1,3,5,7,9];

for (var v of a) {
    console.log( v );
}
// 1 3 5 7 9
```

`for..of`循环请求`a`的迭代器，并自动使用它来遍历`a`的值。

**注意：** 似乎ES6有个奇怪的遗漏，但是普通的`objects`没有像`array`一样的默认*迭代器*。原因超出了本文的范围。如果你只想遍历对象的属性（无法保证特定顺序），`Object.keys(..)`返回一个`array`，该`array`可以用作`for (var k of Object.keys(obj)) { ..`。这种`for..of`遍历对象的键和`for..in`遍历类似，除了`Object.keys(..)`不包含`[[Prototype]]`链的属性，而`for..in`包含。

### Iterables

例子中的`something`对象称为*迭代器*，因为它的接口中有`next()`方法。但另一个紧密相关的术语是*iterable*，它是一个`object`,**包含**能够迭代自己值的*迭代器*。

自ES6起，从`iterable`得到*迭代器*的方法是`iterable`必须有一个函数，名称为特殊的ES6 symbol值`Symbol.iterator`。当这个函数调用的时候，它返回一个*迭代器*。尽管不是必须的，通常每次调用应该返回一个全新的*迭代器*。

前面的`a`是个*iterable*。`fo..of`循环自动调用它的`Symbol.iterator`函数来构建一个*迭代器*。当然，我们也可以手动调用该函数，使用它返回的*迭代器*：

```javascript
var a = [1,3,5,7,9];

var it = a[Symbol.iterator]();

it.next().value;    // 1
it.next().value;    // 3
it.next().value;    // 5
..
```

在之前定义`something`的代码中，你可能注意到这一行：

```javascript
[Symbol.iterator]: function(){ return this; }
```

这段有点混乱的代码是使`something`值--`something`*迭代器*的接口--也是一个*iterable*。之后，我们将`something`传入`for..of`循环：

```javascript
for (var v of something) {
    ..
}
```

`for..of`循环希望`something`是个*iterable*，因此它会寻找并调用`something`的`Symbol.iterator`函数。我们简单地定义函数为`return this`。因此，它只是简单地返回自身，`for..of`循环并不知情。

### 生成器 迭代器（Generator Iterator）

让我们把注意力转回生成器，在*迭代器*背景下。生成器可视作值发生器，通过*迭代器*接口的`next()`调用，我们每次抽取一个值。

因此，从技术上来说，生成器本身不是*iterable*，尽管很像--当你执行生成器的时候，会得到一个*迭代器*：

```javascript
function *foo(){ .. }

var it = foo();
```

我们可以用生成器实现早先的`something`无限数值序列发生器，像这样：

```javascript
function *something() {
    var nextVal;

    while (true) {
        if (nextVal === undefined) {
            nextVal = 1;
        }
        else {
            nextVal = (3 * nextVal) + 6;
        }

        yield nextVal;
    }
}
``` 

**注意：** 正常来说，`while..true`循环包含在JS程序中是一件很糟糕的事，至少在没有`break`或`return`时是如此，因为会永远同步运行，阻塞/锁死浏览器UI。然而，在生成器中，如果有`yield`则完全没关系，因为生成器会在每次迭代时暂停，`yield`回主程序或者事件轮询队列。简单点，“生成器把`while..true`带回了JS编程！”

是不是更简单明了了？因为生成器在每个`yield`处暂停，函数`*something()`状态（域）被保持，意味着整个调用过程中都不需要闭包样版来保存变量状态。

不仅仅代码更简单了--我们不需要实现自己的*迭代器*接口--而且更合理，因为更清晰地表达了我们的意图。例如，`while..true`循环告诉我们生成器打算永远运行--只要我们持续要求，就能够持续生成值。

现在，我们可以使用崭新的采用`for..of`循环的`*something()`生成器。你会发现工作原理基本一致。

```javascript
for (var v of something()) {
    console.log( v );

    // don't let the loop run forever!
    if (v > 500) {
        break;
    }
}
// 1 9 33 105 321 969
```

但别跳过`for (var v of something()) ..`！我们并没有像以前一样简单地引用`something`作为值，而是调用`*something()`生成器来获得*迭代器*供`for..of`循环使用。

如果你密切关注的话，关于生成器和循环的这种交互，可能有两个问题：

+ 为什么我们不能用`for (var v of something) ..`？因为`something`是个生成器，并不是个*iterable*。我们必须调用`something()`来构建一个发生器供`for..of`迭代。
+ `something()`调用生成一个*迭代器*，但是`for..of`循环需要一个`iterable`，不是吗？是的。生成器的*迭代器*也有一个内建的`Symbol.iterator`函数，基本上作`return this`，就像早先定义的`something`*iterable*。换句话说，生成器的*迭代器*也是一个`iterable`！

#### 停止生成器（Stopping the Generator）

在前一个例子中，似乎在循环中的`break`调用之后，`*something()`生成器基本上永远处于挂起状态。

但有个隐藏行为需要你注意。`for..of`“的非正常完成”（即“过早终止”）--通常由`break`，`return` ，或者未捕获的异常引发--会向生成器的*迭代器*发送终止信号。

**注意：** 从技术上讲，在循环正常完成时，`for..of`循环也会向*迭代器*发送这个信号。对生成器而言，本质上是个无意义的操作。因为生成器的*迭代器*必须首先完成，然后`for..of`循环才完成。然而，自定义的*迭代器*可能希望接收到来自`for..of`循环处理过程的附加信号。

尽管`for..of`循环会自动发送该信号，但是你可能希望手动向*迭代器*发送信号，可以通过调用`return(..)`实现。

如果在生成器内部指定`try..finally`，则即使生成器是在外部完成，`try..finally`也总是会运行。如果想清理一下资源（数据连接等等），这很有用：

```javascript
function *something() {
    try {
        var nextVal;

        while (true) {
            if (nextVal === undefined) {
                nextVal = 1;
            }
            else {
                nextVal = (3 * nextVal) + 6;
            }

            yield nextVal;
        }
    }
    // cleanup clause
    finally {
        console.log( "cleaning up!" );
    }
}
```

早先例子`for..of`循环中的`break`会触发`finally`子句。但你可以通过外部`return(..)`手动终止生成器的*迭代器*实例：

```javascript
var it = something();
for (var v of it) {
    console.log( v );

    // don't let the loop run forever!
    if (v > 500) {
        console.log(
            // complete the generator's iterator
            it.return( "Hello World" ).value
        );
        // no `break` needed here
    }
}
// 1 9 33 105 321 969
// cleaning up!
// Hello World
```

当我们调用`it.return(..)`时，它会立马终止生成器，当然会运行`finally`子句。同样的，它也可以向`return(..)`中传入参数来设置返回`value`，那就是`"Hello World"`立即返回的方式。现在我们不需要包含`break`了，因为生成器的*迭代器*被设为`done:true`了，因此`for..of`循环会在下一次迭代时终止。

之所以叫生成器，很大原因要归于处理生成值的用法。但这只是生成器的其中一个用法，坦白来说，甚至不是本书关注的重点。

但既然我们对其工作机制有了更全面的理解，下一步，我们就该把目光转向生成器是如何运用于异步并发的。

## 异步迭代生成器（Iterating Generators Asynchronously）

生成器和异步编程模式有什么关系，修复回调问题之类的？让我们来回到这个重要的问题。

我们重新回顾一下第三章的一个场景。回想一下回调方法：

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

如果想用生成器表示同样的异步流，可以这样：

```javascript
function foo(x,y) {
    ajax(
        "http://some.url.1/?x=" + x + "&y=" + y,
        function(err,data){
            if (err) {
                // throw an error into `*main()`
                it.throw( err );
            }
            else {
                // resume `*main()` with received `data`
                it.next( data );
            }
        }
    );
}

function *main() {
    try {
        var text = yield foo( 11, 31 );
        console.log( text );
    }
    catch (err) {
        console.error( err );
    }
}

var it = main();

// start it all up!
it.next();
```

第一眼看上去，代码有点长，或许比之前的回调更复杂点。但别因第一印象而偏离轨道。生成器版代码实际上更好！但有些东西需要解释一下。

首先看一下这部分代码，是最重要的：

```javascript
var text = yield foo( 11, 31 );
console.log( text );
```

想想这段代码是怎么运行的。我们调用了一个普通的函数`foo(..)`，很明显，我们能够取得Ajax调用返回的`text`，即使它是异步的。

怎么可能？如果你回想下第一章开始的部分，我们有几乎一样的代码：

```javascript
var data = ajax( "..url 1.." );
console.log( data );
```
那段代码（译者注：指第一章开始部分的那段代码）并没有起作用！你注意到不同了吗？生成器中使用了一个`yield`。

那就是神奇所在！那才是允许我们实现看起来是阻塞的、同步的代码，但实际上并不是阻塞整个程序效果的东西；它只是暂停/阻塞了生成器内部的代码。

在`yield foo(11,31)`中，首先`foo(11,31)`被调用，什么都不返回（即`undefined`），因此我们发了个请求来获取数据，但实际上并没有做`yield undefined`。没关系，因为当前代码并没有依赖`yield`的值来做任何有趣的事。我们会在本章后面再讨论这个问题。

此处，我们没有用`yield`来传递信息，只是一种暂停/阻塞的流控制。事实上，在生成器恢复之后，会有信息传递的，但只是单方向的。

那么，生成器在`yield`处暂停，本质上是问个问题，“我应该返回什么值来赋给变量`text`?”谁来回答这个问题呢？

看一下`foo(..)`。如果Ajax请求成功，我们调用：

```javascript
it.next( data );
```

这是以响应的数据恢复生成器，意味着暂停的`yield`表达式直接接收那个值，之后重启生成器代码，之后值赋给局部变量`text`。

相当酷，不是吗？

退一步考虑一下其含义。在生成器内部，我们已经有了看起来是同步的代码（除了`yield`关键词本身），但隐藏在幕后的，`foo(..)`内部，操作可以异步完成。

**意义相当重大！** 回调函数无法以序列化的、我们大脑能关联上的同步方式表达异步，那是针对这一问题的几乎完美的解决方法。

本质而言，我们将异步的实现细节抽象了出来，因此我们可以以同步/序列化的方式解释异步流：“发Ajax请求，当请求完成时，打印出响应”。当然，我们在异步流控制中只表示了两步。但同样的能力可以无界限地扩展，使得我们能够按需表示任意多步。

**提示：** 这是个重要的实现，赶快回去再读读后三段，让它沉入心里。

### 同步错误处理（Synchronous Error Handling）

但是对我们而言，前面的生成器代码有更多的好处。让我们把注意力转向生成器内部的`try..catch`：

```javascript
try {
    var text = yield foo( 11, 31 );
    console.log( text );
}
catch (err) {
    console.error( err );
}
```

这怎么运行的呢？`foo(..)`调用是异步完成的，`try..catch`不是无法捕获异步错误吗，正如第三章中看到的一样？

我们已经看到`yield`是如何让赋值语句暂停来等待`foo(..)`完成的，以至于完成的响应可以赋给`text`。最精彩的部分在于`yield`暂停也允许生成器`catch`错误。我们向生成器中抛入那个错误，如早先代码中的那样：

```javascript
if (err) {
    // throw an error into `*main()`
    it.throw( err );
}
```

生成器的`yield`暂停特性意味着，我们不仅可以从异步函数调用中获取看起来是同步的`return`值，而且也可以从那些异步函数调用中同步地`catch`错误！

既然我们已经看过了如何向生成器中抛出错误，但要是在生成器外也抛出错误呢？正如你所想：

```javascript
function *main() {
    var x = yield "Hello World";

    yield x.toLowerCase();  // cause an exception!
}

var it = main();

it.next().value;            // Hello World

try {
    it.next( 42 );
}
catch (err) {
    console.error( err );   // TypeError
}
```

当然，我们可以通过`throw..`手动抛出错误，而不是引发一个异常。

我们甚至可以捕获像`throw(..)`进生成器一样的错误，本质上是给生成器一个机会来处理它，但如果不处理，*迭代器*代码必须处理：

```javascript
function *main() {
    var x = yield "Hello World";

    // never gets here
    console.log( x );
}

var it = main();

it.next();

try {
    // will `*main()` handle this error? we'll see!
    it.throw( "Oops" );
}
catch (err) {
    // nope, didn't handle it!
    console.error( err );           // Oops
}
```

就可读性和可推理性而言，看起来同步的错误处理（通过`try..catch`）实现异步错误处理是个巨大的胜利。

## 生成器 + Promise（Generators + Promises）

在之前的讨论中，我们展示了如何异步迭代生成器，相对于意大利面条式的回调混乱，序列化的可推理性是个巨大的进步。但是我们丢了一些非常重要的东西：Promise的可信任和可组合性（见第三章）！

别担心--我们会找回来的。ES6中最棒的部分是组合Promise和生成器（看起来同步的异步代码）。

但怎么做呢？

回想下第三章中基于Promise的Ajax例子：

```javascript
function foo(x,y) {
    return request(
        "http://some.url.1/?x=" + x + "&y=" + y
    );
}

foo( 11, 31 )
.then(
    function(text){
        console.log( text );
    },
    function(err){
        console.error( err );
    }
);
```

早先的Ajax生成器代码中，`foo(..)`什么都不返回（`undefined`），*迭代器*控制代码不关心`yield`的值。

但Promise式的`foo(..)`在Ajax调用后返回了一个promise。表明我们可以用`foo(..)`构建一个promise，之后从生成器中`yield`出来。

但*迭代器*如何处理promise呢？

它应该监听promise解析（fulfillment或者rejection），之后既可用fulfillment信息恢复生成器，又可用错误原因向生成器中抛出错误。

让我重复一遍，因为很重要！获取Promise和生成器精华最自然的方式是`yield`**出一个Promise**，连接那个Promise来控制生成器的*迭代器*。

我们来试一下！首先，我们将生成器`*main()`和Promise式的`foo(..)`放在一起：

```javascript
function foo(x,y) {
    return request(
        "http://some.url.1/?x=" + x + "&y=" + y
    );
}

function *main() {
    try {
        var text = yield foo( 11, 31 );
        console.log( text );
    }
    catch (err) {
        console.error( err );
    }
}
```

这个重构最强大的启示是`*main()`中的代码**一点也不需要变！**在生成器内部，`yield`出什么值是不透明的实现细节，因此我们甚至没有意识到它的发生，也不需要去担心了。

但现在，我们该如何运行`*main()`？仍然有一些实现探究工作要做，接收和连接`yield`出的promise，以便一旦解析，就恢复生成器。我们手动试下：

```javascript
var it = main();

var p = it.next().value;

// wait for the `p` promise to resolve
p.then(
    function(text){
        it.next( text );
    },
    function(err){
        it.throw( err );
    }
);
```

事实上，一点也不痛苦，不是吗？

这段代码很像我们早先做的，即手动连接的由错误优先回调控制的生成器。promise已经为我们分成fulfillmeng（成功）和rejection（失败），而不是`if (err) { it.throw..`，但除此之外，*迭代器*控制是一样的。

现在，我们已经掩盖了一些重要的细节。

最重要的是，我们利用这一事实，即我们知道`*main()`内部只有一个Promise式的步骤。要是我们想用Promise驱动生成器，不管它有几步呢？我们当然不想为每个生成器手动写不同的Promise链！如果有一种方法重复（即“循环”）迭代控制，每次返回一个Promise，在继续之前等待其解析结果就好了。

另外，要是在`it.next(..)`调用过程中，生成器抛出一个错误（有意的或者无意的）呢？我们是该停止，还是`catch`它并把它发送回去？同样的，要是我们`it.throw(..)`一个Promise rejection到生成器中，但没有被处理，又被抛出来了呢？

### Promise-Aware Generator Runner

沿着这条道路探索越多，你就越会意识到，“哇哦，如果有utility给我用就太棒了。”你问对了。这是一种很重要的模式，你不想把它弄错（或者耗尽精力地一遍一遍重复），因此你最好的赌注是使用一个utility，专门用来运行Promise-以我们所举的方式`yield`出生成器。

有几个Promise抽象库提供了这样的utility，包括我的*异步序列*库和它的`runner(..)`，会在本书的附录A中讨论。

但为了学习和说明，让我们简单定义下单独的utility，叫做`run(..)`：

```javascript
// thanks to Benjamin Gruenbaum (@benjamingr on GitHub) for
// big improvements here!
function run(gen) {
    var args = [].slice.call( arguments, 1), it;

    // initialize the generator in the current context
    it = gen.apply( this, args );

    // return a promise for the generator completing
    return Promise.resolve()
        .then( function handleNext(value){
            // run to the next yielded value
            var next = it.next( value );

            return (function handleResult(next){
                // generator has completed running?
                if (next.done) {
                    return next.value;
                }
                // otherwise keep going
                else {
                    return Promise.resolve( next.value )
                        .then(
                            // resume the async loop on
                            // success, sending the resolved
                            // value back into the generator
                            handleNext,

                            // if `value` is a rejected
                            // promise, propagate error back
                            // into the generator for its own
                            // error handling
                            function handleErr(err) {
                                return Promise.resolve(
                                    it.throw( err )
                                )
                                .then( handleResult );
                            }
                        );
                }
            })(next);
        } );
}
```

如你所见，





















