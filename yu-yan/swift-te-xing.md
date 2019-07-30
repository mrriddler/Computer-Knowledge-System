# Swift特性

Swift是一门快速、安全、交互式的语言。快速指的是**依靠LLVM优化过后的机器码运行时效率非常高**。

安全指的是：

* **静态安全，让编译器检查出更多的潜在错误**，而这依赖于Swift强大的类型系统\(可能也是导致编译慢的罪魁祸首\)。
* **写出更可预测\(predictable\)的代码**，这依赖于Swift引入的概念与设计。
* **难度更高的逆向**。

**交互式指的是非常有表现力\(expressive\)的语法，代码会有更好的维护性**，比如Swift中的if和switch…case可以跟随where进行模式匹配。

## 范式

Swift是一个结合多范式特性的语言，包括面向协议编程\(POP\)、面向对象编程\(OOP\)、函数式编程\(FP\)、泛型编程\(Generic\)，Swift则是希望把这些范式结合出一加一大于二的效果。

Swift革命之初就提出了面向协议编程\(POP\)。其世界观是：一切都是性质构成的主体，性质可以构成复合性质，而由性质构成主体，主体具有状态和行为，状态通过行为联系起来，构成整个世界。面向对象编程更像是面向名词抽象，而面向协议编程更像是面向形容词抽象。

POP是Swift设计优雅的基石。将Swift的基础库\(Sequence、Collection\)对比其他语言的基础库很容易可以看出来，比如说Java，Swift的优雅跃然于IDE上。Sequence抽象出广义的序列，Collection抽象出具有数据结构共同性质的序列，而具体的数据数据结构，交由具体的序列实现。Swift在POP范式下，抽象的更加纯粹、单一。

OOP和FP之间的擦碰必将会迸发出不一样的火花。很多语言将函数式仅作为一个支持，而不是结合。Swift可不仅仅是为语言添加closure这么简单，Swift真真正正是一门函数式\(functional\)语言，FP影响了语言的方方面面。这一点上，Scala是结合OOP和FP特性的开拓者，Swift也从中借鉴了很多。

## 类型编程

> Both existential types and generics depend on dynamic dispatching based on protocols. A value of an existential type \(say, Comparable\) is a pair \(value, vtable\). 'value' stores the current value either directly \(if it fits in the 3 words allocated to the value\) or as a pointer to the boxed representation \(if the actual representation is larger than 3 words\). By itself, this value cannot be interpreted, because it's type is not known statically, and may change due to assignment. The vtable provides the means to manipulate the value, because it provides a mapping between the protocols to which the existential type conforms \(which is known statically\) to the functions that implements that functionality for the type of the value. The value, therefore, can only be safely manipulated through the functions in this vtable.
>
> A value of some generic type T uses a similar implementation model. However, the \(value, vtable\) pair is split apart: values of type T contain only the value part \(the 3 words of data\), while the vtable is maintained as a separate value that can be shared among all T's within that generic function.
>
>  --Swift Generics Doc

Swift依靠强大的类型系统有自己一套独特的类型编程。Swift主范式面向协议编程把协议与类看作同等的地位，协议可作为一个独立的类型，即**存在类型\(existentials type\)**。而协议变量被包裹在**存在容器\(existentials\)**中。

### **实现**

面向对象编程调用方法一共就两种种方式：动态发消息\(dynamic dispatch\)、静态调用。而**存在类型**调用函数实现的思路就类似**dynamic dispatch**。这是由编译期和运行时协作实现的。

编译期会生成每个协议和具体类型的函数列表，并会为协议变量生成**存在容器**包括其相关代码。容器中有一些元数据和类型转换信息，其中最重要的就是vtable。vtable放置具体类型实现协议的函数。运行时，变量再跳转到vtable调用函数。

除此之外，如果是协议拓展的、不需要具体类型实现的函数，是不通过**dynamic dispatch**，而直接静态调用。

### **协议泛型**

由于语义上和实现上的限制，协议泛型根本不成立。Swift不会有协议泛型，反倒是Swift发明了关联类型\(associatedtype\)和类型别名\(typealias\)来抽象协议中的类型，实现一定程度的协议类型抽象。可惜的是包括associatedtype和Self的协议还没法作为存在类型，这点还没能实现出来。现有的方法就是类型擦除\(type erased\)绕过这个限制。

Swift提供的协议类型抽象编程有以下几点：

* 关联类型\(associatedtype\)：协议可以关联类型。当主体遵守协议的时候，既可以直接指明关联类型，也可以隐式地交由编译器推断。
* 类型别名\(typealias\)：给已有类型取别名。
* 类型本身\(Self\)：代表类型本身。

### **泛型**

Swift对泛型的态度：[作为代码复用、加强类型安全、减少类型转换的引入](https://github.com/apple/swift/blob/master/docs/Generics.rst)，完全不支持泛型元编程。

### **存在泛型**

**存在泛型\(existential generics\)**与**存在类型**本不是一个系统，但由于其都是重度依赖协议，所以交错在了一起。

Swift实现的**存在泛型**并不是像C++模板纯靠编译器，同样是通过**dynamic dispatch**实现的，编译器和运行时协作实现。Swift不完全依赖于编译器生成是因为让任何修改泛型函数签名的行为都导致重新全量编译，防止像C++那样编译慢的噩梦。并且也有利于减少二进制大小。与**存在类型**不同的是，泛型的vtable不是针对变量而是针对泛型类型的。

### **泛型约束**

泛型约束是Swift类型编程中的主要一环，泛型约束指明类型或其内部的子类型必须遵守的规则。Swift只支持函数泛型约束，而不支持struct或class的泛型约束。

### **特化**

泛型约束实质上就是特化，Swift对特化有较好的支持。

Swift还有个关于特化的属性@\_\_specialize，这实际上只是个对编译优化器的一个提示，尝试绕过**dynamic dispatch**，直接在编译期内联。不影响函数的泛型签名，对实际特化没影响。

它有两个指定参数：

* exported：可选值true、false，代表控制权限是否为public
* kind：可选值full、partial，代表是否是全部匹配

指定参数后，紧跟where语句标识匹配主体的声明，比如：

```swift
@_specialize(exported: true, kind: full, where K == Int, V == Int)
```

### **元类型**

通过调用类型的.Type，可以获得该类型的类型，.self可以获得该类型本身。有了元类型和类型本身，可以一定程度的元编程。

## 求值

### let和var

Swift中将所有数据声明分为两种类型：

* 不可变数据\(let\)
* 可变数据\(var\)

OOP和FP适用不同的求值模型，函数代换模型和环境求值模型，而要想要在同一求值模型下计算，计算机科学给出的解答就是在环境求值模型下提供不可变数据和可变数据。要想在语言中将这两种范式作为平级，合适的做法就是要让任意数据都可以区分不可变数据和可变数据。Swift直接从数据声明入手，这是从Scala借鉴过来。

### value type和reference type

将所有变量分为两种类型：

* 值类型\(value type\)
* 引用类型\(reference type\)

这两种类型即有语义的区别，也有内存管理的区别。

从语义上讲，值类型只能有被一方拥有\(own\)，引用类型可被多方引用。使用值类型可以写出更加可预测的代码，任何对值类型的改变都在可控的范围上下文内。

从内存管理上讲，值类型当被赋值、传递的时候，会复制出另一个实例去操作。而引用类型在相同情况下，仍是同一个实例。值类型还有以下两点优化：

* 对集合类结构体的copy on write：对于集合类结构体，其内部指向的是同个内存地址，直到有一方改变。除集合类结构体没有这种优化。
* 堆存储转移到栈存储：情况允许，将结构体优化到栈上存储。

### Algebraic data type

在函数式编程中或者说类型编程中，存在这样一种角度去看待类型，将所有类型都[看作成代数](https://en.wikipedia.org/wiki/Algebraic_data_type)，可以进行代数运算。比如，对于enum，因为enum最多可能包含的类型就是其case的和，所以叫做sum type\(和类型\)。同理，tuple和struct则是product type\(乘类型\)。

Swift提供的enum，有自己的类型，可以关联\(associate\)其他值，可以具有函数。真正有意思的是将enum声明为indirect可以有递归性质。这就代表有递归性质的数据可以使用enum抽象，这就包括大部分树状数据结构，其抽象出来的代码相当优雅。

## Optional

Swift除了在泛型编程上体现出用强大的类型系统保证安全，还发明了optional。optional就是一个值的枚举wrapper，可能有值，也可能没有，就像薛定谔的猫一样。而为了快速unwrap optional，Swift还提供了一些语法糖，包括optional binding、optional chainning等。optional的出现自然也帮助开发者写出更加可预测的代码。

## 引用

[http://condor.depaul.edu/ichu/csc447/notes/wk10/Dynamic2.htm](http://condor.depaul.edu/ichu/csc447/notes/wk10/Dynamic2.htm)

[https://objccn.io/products/advanced-swift/](https://objccn.io/products/advanced-swift/)

