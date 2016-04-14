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
















