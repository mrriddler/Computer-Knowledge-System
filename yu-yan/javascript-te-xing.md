# JavaScript特性

JavaScript不是一门孤独的语言，它影响了很多语言。以ECMAScript作为标准实现，影响了很多其他基于ECMAScript的语言，包括JScript和ActionScript等。而JavaScript又作为一门编译目标语言，还给JavaScript带来了衍生语言，包括TypeScript等。

> 本文更新至ES6。

JavaScript是一个矛盾体，JavaScript架设在很多偏离工业界的理论基础。但是，作为工业语言，JavaScript并没有追求纯粹，JavaScript做了很多妥协。如果用一句话形容JavaScript，那就是用一层大家都熟悉的面孔盖住了骨子里的陌生。这样不会一下子吓跑的别人，但会让人经常在意想不到的地方，迈入陷阱。

> JavaScript的许多特性都借鉴自其他语言。语法借鉴自Java，函数借鉴自Scheme，原型继承借鉴自Self。而JavaScript的正则表达式则借鉴自Perl。
>
> — JavaScript语言精粹

## 范式\(原型\)

JavaScript是一门基于原型的语言，它也是基于原型中最广泛使用的语言。原型也可以叫原型子系统，这指的是基于原型是面向对象的一个分支，另一个常见的分支就是基于类\(class\)了。

原型的英文名是prototype，而这个prototype与设计模式中的原型模式同名。实际上，设计模式prototype就是在非基于原型中，实现原型。用一句恰当的话来描述：基于原有的形状进行“有丝分裂”。

原型在做“分裂”的时候，有两种方式，一种是委托、一种是关联。委托不复制原型，新“分裂”出来的物体保有对原型的链接，即原型可以对物体产生副作用。关联，则复制原型，复制后则与原型不相干，即原型不会对物体产生副作用。JavaScript则是委托。

基于原型与基于类一个重要的区分就是：静态和动态。静态指的是，在编译时确定对象的状态和行为，形成编程者之间的“约定”。而动态没有这样的“约定”，在运行时再确定对象的状态和行为。同样是面向对象编程，但区分出了两种编程模型。

而没有了这样的“约定”，也就没有检查这些“约定”的必要。伴随基于原型的常常是解释型、弱类型。

JavaScript遵循ECMAScript标准实现prototype，主要是折腾[instance的proto和function的proto、prototype](https://github.com/creeperyang/blog/issues/9)：

* 不论instance还是function作为对象，其proto指向其继承体。不过function即作为Function的instance，又能作为constructor创建其他instance，多了prototype。而function的prototype是个独立的个体，就是function.prototype。
* 当instance被new的时候，其proto指向其constructor的prototype。
* 同理，function由Function构造，其proto为Function.prototype。
* 所有instance的proto最终都会顺藤摸瓜到Object.prototype，包括Function.prototype，而Object.prototype的proto为null。

## 环境和求值

JavaScript存在非常多令人匪夷所思的行为，其中环境和求值是其中的大头。

JavaScript是一门C style语法语言\(参照的Java\)，而提到C style，第一反应往往是块级作用域，但JavaScript却不支持块级作用域。JavaScript只有两种作用域，全局作用域和函数作用域。函数作用域除了在函数内，还包括with、try、catch。

### 环境

在JavaScript中，环境求值模型基于作用域链来实现：

* 定义函数的时候，将当前作用域链中的head作用域链接到函数的\[\[Scope\]\]。
* 调用函数的时候，将函数的\[\[Scope\]\]链接到作用域链head，并结合活动对象\(Activation Object\)，求值函数。

这两条除了实现了求值模型，还实现了词法作用域。

### 变量提升

在function里面无论在哪个位置声明的var，都相当于声明在起始位置。这实际上是求值器的一种变种，这种变种会先将所有声明的变量提升到顶部，另一种就是经常接触的顺序求值，也就是先声明再使用。

### this

this作为当前环境的抽象，实际上与其他语言是一致的。不过，其他语言在作用域的影响下，无论是this还是self基本只出现在method中，造成了this和self是指向Class对象的假象。实际上，在任何语言中，this或self都代表当前环境。不好理解的是，JavaScript调用函数的方式会影响所在的具体环境，进而也就影响了求值this。然而更繁琐的是，JavaScript甚至在用strict、箭头\(=&gt;\)、new都会影响求值this。

## 模块化

一般语言模块化，引入会将其他模块整体引入。而JavaScript在作用域的影响下，这样做会使全局命名空间污染成为家常便饭。JavaScript反其道而行之，将每个文件单做一个解释单位，天然的隔离开不同单位。而单位要被引用时，必须指明导出的内容。引用者一般也要指明需引用的内容。这就形成了JavaScript独特的引入、导出方式，且相较其他语言，引入和导出还有较为复杂的规则。

这种引入方式还有很多变种。

* 静态引入：编译时引入。
* 动态引入：运行时引入。
* 值引入：引入的是复制过的值。
* 引用引入：引入的是引用本身。

仔细推断一下，可以得出结论，静态引入会和引用引入结合起来才有意义，而动态引入要和值引入。第一种则是import，第二种是require。

## 并发程序设计

JavaScript中的并发程序设计最早的就是yield，yield背后的概念是协程，协程有一个兄弟子程。协程和子程是程序的一个关键模块化方式。

* 子程：分为主程序和子程序，有主次关系。
* 协程：程序之间是平行的关系，没有即定的调用顺序。

子程序可以这样描述：

![](../.gitbook/assets/subroutine.png)

协程可以这样描述：

![](../.gitbook/assets/coroutine.png)

### 入口和出口

> Subroutines are special cases of ... coroutines.
>
> — [Donald Knuth](https://en.wikipedia.org/wiki/Donald_Knuth)

祖师爷说可以将子程看做是一种特殊的协程，可以从入口和出口来理解这一点。子程很简单，主程序从入口进入，调用每个子程序后，从出口出去。而协程就不一样了，每次外部发送信号\(next/send\)，协程都是进入或再次进入入口，然后自由地将任何位置作为出口出去。这样来看，子程就是全自动进入的协程。

### 角色

在协程中有这样的三种角色：

* Producer：生产者，只调用send/next。
* Filter：转换器，调用send/next和yield。
* Consumer：消费者，只调用yield。

那么，这三种角色在执行流中可以这样描述：

![](../.gitbook/assets/produce_filter_consume.png)

### generator

generator是协程的子集，generator可以叫做[半协程\(semicoroutines\)](https://en.wikipedia.org/wiki/Coroutine#Comparison_with_generators)，半协程与协程不同的地方就是在于半协程不能自由出去，只能是遇到yield作为出口。这从一定程度上简化了协程的操作和隐藏了协程的概念。

从generator取名来看，和producer有异曲同工之妙。

### async/await

async/await最早是在[C\#中提出的](https://msdn.microsoft.com/en-us/library/hh191443%28v=vs.120%29.aspx)，同样为简化并发程序设计而诞生，发展到现在，async/await已成了GUI系统异步编程的标配。

## 引用

JavaScript语言精粹

[https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/this](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/this)

[http://es6.ruanyifeng.com/\#docs/intro](http://es6.ruanyifeng.com/#docs/intro)

[http://weizhifeng.net/javascript-the-core.html\#activation-object](http://weizhifeng.net/javascript-the-core.html#activation-object)

[https://wizardforcel.gitbooks.io/sicp-in-python/content/29.html](https://wizardforcel.gitbooks.io/sicp-in-python/content/29.html)

