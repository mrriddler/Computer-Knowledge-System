# 编程漫游

## 有趣的事实\(Fun-Fact\)

```text
printf("hello, world\n");              \\C
(display "hello, world")               \\Scheme
NSLog(@"hello, world\n");              \\Objective-C
print("hello, world\n")                \\Swift  
std::cout << "hello, world\n";         \\C++
System.out.println("hello, world\n");  \\Java
print 'hello, world'                   \\Python
console.log("hello, world");           \\JavaScript
```

有没有好奇过，为什么每学一门编程语言，第一个要写的程序一定是打印hello， world？这得追溯到[The C Programming Language](https://www.amazon.com/Programming-Language-Brian-W-Kernighan/dp/0131103628)这本书，这本书在1978年发行第一版，作为世界上第一本教授编程语言的书。而书上的第一段程序就是打印hello world。这本书很可能是大部分语言的作者看的第一本编程书，没有这本书，也许就没有计算机语言的大家庭。所以，每个程序员为了祈求程序安稳运行，都要在学习语言的第一段程序写下hello，world这样的咒语。

本篇文章在“悟”出抽象形成隔离和推向更广泛的基础上，着眼于用计算机语言抽象的门道。从概念出发，融汇到常见的程序中，再次反思其本质。

## 抽象门道两大要素

抽象门道两大要素：

* 数据抽象：将表达数据的表达式定义为数据。
* 过程抽象：将计算片段定义为过程。

用计算机语言抽象有这两个抽象，这两大要素也独立于范式、语言。在编写程序时，功力越深，越能将这两大要素抽象的恰如其分，达到一个管理软件复杂度的目的。计算机软件工程相比其他工程有自己的特点，其中最有意思的就是，能在极短的时间内推翻工程并再造，这就带来了始料不及的需求变化。而在快速的变更中继续壮大工程，是软件工程的核心挑战。要去挑战这条恶龙，开发者的宝剑就是管理软件的复杂度。而通常的手段就是要选取合适的抽象层级，形成模块化。

## 过程抽象

过程抽象的本意将特定的表达式推向成更广泛的过程，过程还将数据与如何将数据应用于过程隔离开来。

### 高阶函数\(higher-order-function\)

将特定的过程抽象成更广泛的过程，就引出了高阶函数。高阶函数将过程应用于过程，过程也可以返回过程，以提高抽象层级，提高表达能力。

### **通用接口\(conventional-interface\)**

通用接口是最常见的高阶函数，包括，map、for-each、filter、fold-right\(accumulate\)、fold-left\(reduce\)、append、flatten、flatmap等等，它们是任意数据都可以提供的高阶函数接口\(比较常见的是线性数据结构中\)，这些接口通过高阶函数的方式，可以提供给数据很强的表达能力。并且还可以将这些接口链\(chainning\)调用形成流水线\(pipeline\)，以分解复杂操作\(operator\)，使每个过程单一化。以下是Scheme中，如何实现这些通用接口。

```scheme
(define (map op sequence)
        (if (null? sequence)
            nil
            (cons (op (car sequence))
                  (map op (cdr sequence)))))
​
(define (for-each op sequence)
        (cond ((not (null? sequence))
              (op (car sequence))
              (for-each op (cdr sequence)))))
​
(define (filter predicate sequence)
        (cond ((null? sequence) nil)
              ((predicate (car sequence))
                          (cons (car sequence)
                                (filter predicate (cdr sequence))))
              (else filter predicate (cdr sequence)))))
​
(define (fold-right initial op sequence)
        (if (null? squence)
            initial
            (op (car squence)
                (fold-right initial op (cdr sequence)))))
​
(define (fold-left op initial sequence)
        (define (iter result rest)
                (if (null? rest)
                    result
                    (iter (op result (car rest))
                          (cdr rest))))
        (iter initial sequence))
​
(define (append list1 list2)
        (if (null? list1)
             list2
            (cons (car list1)
                  (append (cdr list1) list2))))
​
(define (flatten sequence)
        (fold-right nil append sequence))
​
(define (flatmap op sequence)
        (flatten (map op sequence)))
```

## 数据抽象

数据抽象与过程抽象相近，过程抽象将特定的表达式隔离出来，使过程更加广泛。而数据将如何使用数据和数据本身如何构造隔离开来，使数据更加广泛。

### 构造数据

对数据做抽象，经常需要将多个信息“粘\(glue\)”起来。元组\(tuple\)就是构造数据的最基本手段。

由二元组可以构造出一切数据，让每个节点的第一个元素代表节点的值，第二个元素代表与下一节点的连接。二元组就构造出了线性数据结构。

```text
(value, (value, (value, (value, nil))))
```

让节点的第一个元素代表节点的值，第二个元素用线性数据结构代表节点的任意子节点。二元组也就构造出了树状数据结构。

```text
(value, 
       (value, nil)
)
​
(value, 
       ((value, nil),
       ((value, nil),
        (value, nil)))
)
```

既然可以构造出线性数据结构和树状数据结构，二元组自然也可以构造出图状数据结构和集合。

元组甚至可以由一系列两种类型的过程组成的，构成\(constructor\)和选择\(selector\)。在Scheme中，可以使用`cons`、`car`、`cdr`过程构造出二元组，`cons`为构成类型过程。`car`取出二元组的第一个值，`cdr`取出二元组的第二个值，`car`、`cdr`为选择类型过程。

```text
(define (cons x y)
              (define (dispatch m)
                      (cond ((= m 0) x)
                            ((= m 1) y)
                            (else (error "Argument not 0 or 1 -- CONS" m))))
               dispatch)
​
(define (car z) (z 0))
​
(define (cdr z) (z 1))
```

元组可以构造一切数据，又由一系列过程组成。这表明数据与过程之间没有一条很清晰的界限。不过，就理解编程来说，还是不要把它们混为一谈。

## 状态

当变量改变了其值，变量便有了状态，这背后是时间维度的介入。**变量值的改变即状态改变隐式的反映出时间的流逝**。只要有了时间，编程的时候就需要注意状态变量的语句顺序，语句顺序会影响程序的正确性，这是个不小的麻烦。

### 可变数据\(mutable\)与不可变数据\(immutable\)

为了避免时间带来的麻烦，就衍生出了可变数据和不可变数据。对于可变数据，存在状态，改变数据后还是同一个数据，数据只是改变了状态。而不可变数据，不存在状态。改变数据后，改变后的数据和未被改变时的数据不是同一个数据。

### 惰性求值\(lazy-evaluation\)

计算机计算值有两种方法，随机访问\(random-access\)和惰性求值，随机访问将数据先计算好放入存储器，访问数据时，直接访问存储器。而惰性求值，顾名思义，不会将数据计算好放入存储器，访问数据时再进行计算。惰性求值有两种变种：

* 缓存值：缓存来提升整体计算性能。
* 不缓存值：每次访问都重新计算值，这样虽然相比缓存性能不佳，但可实现不可变数据。

惰性求值既可以由求值模型的应用序和正则序来决定，也可以由如何编写程序决定。惰性求值在Haskell语言中是默认的求值方式，这是由求值模型决定的，那么其他无法改变求值模型的语言要靠流来实现惰性求值了。

### 流\(stream\)

流是惰性求值的线性数据。流的性质是，只构造出了流的一部分，如果使用者需要使用流未构造出的部分，流就自动构造下去，像窗口一样，“流动”下去，这也就是为什么被命名为流。这样，使用流的时候就制造出了整个流都已经构造出来的假象，那么流也可以是无限的了。流保存如何计算值\(lambda\)，而不直接保存值。这样流就由流的值和如何计算下一个流构成。如何计算下一个流要先通过此流的值计算出下一个流的值，再将计算出的值和定义出的如何计算下下一个流构成新的流。以下就是Python简单实现出的Stream。

```python
class Stream(object):
        def __init__(self, first, compute_rest, empty=False):
            self.first = first
            self._compute_rest = compute_rest
            self.empty = empty
            self._rest = None
            self._computed = False
        @property
        def rest(self):
            """Return the rest of the stream, computing it if necessary."""
            assert not self.empty, 'Empty streams have no rest.'
            if not self._computed:
                self._rest = self._compute_rest()
                self._computed = True
            return self._rest
        def __repr__(self):
            if self.empty:
                return '<empty stream>'
            return 'Stream({0}, <compute_rest>)'.format(repr(self.first))
```

以下是如何创建一个Stream。

```python
s = Stream(1, lambda: Stream(2+3, lambda: Stream.empty))
```

每个Stream都由流的值和lambda表示的如何计算下一个流构成。

### 响应式编程\(FRP\)

拓展此概念，将所有数据都由流来创建，就诞生了响应式编程范式。在响应式编程中，一切都是无穷无尽的流。

举个简单的例子:

```text
命令式：
int i = 0
++i
++i
​
响应式：
Observable->onNext(i=0)->onNext(i=1)->onNext(i=2)
subscribe->i
  
对于命令式中，i是可变数据，存在状态，如果你想在i=1的时候，做一些事，要注意语序。
而响应式中，i是流，不存在状态，完全不需要注意语序。
```

在这个例子中，除了展示了一切都是流以外，还展示了命令式编程语言就是在告诉如何计算机如何做\(how to do\)，而FRP告诉计算机做什么\(what to do\)，这就是FRP是声明式范式的子范式的一种表现。

## 引用

计算机程序的构造和解释\(SICP\)--原书第二版

SICP in python [https://wizardforcel.gitbooks.io/sicp-in-python/content/index.html](https://wizardforcel.gitbooks.io/sicp-in-python/content/index.html)

FRP [http://reactivecocoa.io/philosophy.html](http://reactivecocoa.io/philosophy.html)

