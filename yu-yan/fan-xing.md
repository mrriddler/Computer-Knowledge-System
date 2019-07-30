# 泛型

泛型是一种主范式的补充\(complementary\)范式，对类型做抽象，解决由于类型带来的代码复用问题、加强类型安全、减少类型转换等问题。诞生之初，泛型是个对OOP的补充\(complementary\)。这个补充主要指的是如果用OOP的继承解决复用问题，会导致很明显的问题，完全违反了组合优于继承的设计原则，所以泛型代替继承去解决复用问题。从泛型能做的事或者说能解决问题的角度来看，泛型不是主范式的料，也就作为为主范式的补充范式。

## 编译时多态

泛型编程与面向对象编程的特征多态本意是一样的，都是根据不同的类型表现出不同的“形状\(shape\)”。所以泛型编程又叫做参数化类型\(parameterized type\)或者参数化多态\(parametric polymorphism\)。而相对于面向对象编程运行时确定类型，泛型中的参数化类型都是在编译期确定的，所以泛型编程又叫做编译时多态，面向对象编程的特征多态又叫做运行时多态。注意，虽然说是在编译器确定下来，但是不代表一定要由编译器实现。

## 特化和偏特化

被抽象后的事物就像人一样，总想要表现出自己独特一面。实际上就是多态的意思，而在泛型中，这种将泛化的东西叫specialization，就是特化，只特化一部分就是偏特化。

## Constraints

constraints是泛型用来实现特化和偏特化的一个重要概念，要实现特化，就需要为重载的函数或类型添加一个假设，决议的时候，那个特化使假设成立了，就会选择这个特化。

C++中决议这个行为叫做Concepts Lite，而约束叫做Concepts。

[ISO中有篇提案关于C++](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4553.pdf)的Concepts是这样描述的：

> the specification and checking of constraints on template arguments, and the ability to overload functions and specialize class templates based on those constraints.

## 不同语言中的泛型

有意思的是，不同语言根据自己的需要对泛型有不同程度的支持，比如像C++甚至支持到元编程的地步，[而像Swift是作为代码复用、加强类型安全、减少类型转换的引入](https://github.com/apple/swift/blob/master/docs/Generics.rst)，完全不支持元编程。而Java和Objective-C，支持泛型就比较弱，Objective-C更是一种轻量泛型，就是给编译器的一个提示，没有实际的作用。

### C++ Template

在C++中，泛型叫做template。template的语法玄幻，再加上泛型中形参、实参、实例化、元编程，使得template完全可以单独当做一门语言来学习，再加上大部分由编译器实现导致的行为不透明。使得template着实成为了一个难点。

以下代码表达了通过enable\_if实现特化，使编译器可以决议出具体使用的AnyType是哪个类型。

```cpp
enum class Type: int {
    Null,
    String,
    Numberic,
};
​
template<Type T = Type::Null>
struct TypeInfo {
    static constexpr const bool isNull = true;
    static constexpr const bool isString = false;
    static constexpr const bool isNumberic = false;
    static constexpr const Type type = Type::Null;
    static constexpr const char *name = "Null";
};
​
template<>
struct TypeInfo<Type::String> {
    static constexpr const bool isNull = false;
    static constexpr const bool isString = true;
    static constexpr const bool isNumberic = false;
    static constexpr const Type type = Type::String;
    static constexpr const char *name = "String";
};
​
template<>
struct TypeInfo<Type::Numberic> {
    static constexpr const bool isNull = false;
    static constexpr const bool isString = false;
    static constexpr const bool isNumberic = true;
    static constexpr const Type type = Type::Numberic;
    static constexpr const char *name = "Numberic";
};
​
template<typename T, typename Enable = void>
struct IsStringType : std::false_type {
};
​
template<typename T>
struct IsStringType<
        T,
        typename std::enable_if<std::is_same<std::string, T>::value ||
                 std::is_same<const char *, T>::value>> : std::true_type {
};
​
template<typename T, typename Enable = void>
struct IsNumbericType : std::false_type {
};
​
template<typename T>
struct IsNumbericType<
        T,
        typename std::enable_if<std::is_integral<T>::value ||
                 std::is_enum<T>::value ||
                 std::is_floating_point<T>::value>> : std::true_type {
};
​
template<typename T, typename Enable = void>
struct AnyType : public TypeInfo<Type::Null> {
};
​
template<typename T>
struct AnyType<
    T,
    typename std::enable_if<IsStringType<T>::value>::type> : TypeInfo<Type::String> {
};
​
template<typename T>
struct AnyType<
    T,
    typename std::enable_if<IsNumbericType<T>::value>::type> : TypeInfo<Type::Numberic> {
};
```

如果上文中是你初次接触到enable\_if，可能会觉得它很magic。实际上，enable\_if的核心是SFINAE\(Substitution Failure Is Not An Error\)原则，话句话说就是，编译器不将替代失败当成错误处理。当调用重载函数或类型的时候，编译器会从所有可能的函数或类型候选中，挨个决议是否为最合适的，如果出现了不合适的，即替代失败的，就会从候选中划除，不会直接报错。如果最后没能找到任何候选，编译器就会报出编译问题。以下是几点C++编译器应对泛型的行为：

* 泛型实例化后，编译器生成出该实例化类型的函数或类型的代码。
* 对于函数重载，泛型会融入函数签名，决议时直接匹配函数签名。
* 使用泛型的时候，编译器可以根据上下文进行类型推导。你也可以指明部分类型，部分留给编译器推导，只不过指明的部分要声明在前。
* 众所周知，编译器进行语法、语义分析的时候，要想明白写下的字符串的意思，就要进行name lookup。这事遇上泛型就更有意思了，在泛型没有实例化之前，name lookup是得不到正确结论的，不知道实际的类型，自然就不知道使用的变量或函数是否正确。所以，name looup遇上泛型，就会分为两大类，依赖泛型和不依赖泛型，不依赖泛型会直接进行name lookup，而依赖泛型的需要等到泛型实例化的时候再进行name lookup，这也被称为two phase look up。

### **Swift**

Swift实现的泛型有些独特，因为Swift不是将泛型作为面向对象编程的补充范式，而要作为面向协议编程的。Swift就发明了关联类型\(associatedtype\)和类型别名\(typealias\)来抽象协议中的类型，实现协议泛型。

而在Swift中，特化可以由模式匹配：或where来表达。

Swift实现的泛型并不是像C++那样，泛型都是通过dynamic dispatch实现的，通过编译器和运行时配合实现的，做到根据类型动态派发。Swift不完全依赖于编译器生成是因为让任何修改泛型函数签名的行为都导致重新全量编译，防止向C++那样编译慢的噩梦。并且也有利于减少二进制大小。

Swift最独特的特性Optional就是由泛型实现的，泛型的类型实际上充当了Wrapper，而泛型为Wrapped，泛型的巧妙就在于可以让Wrapper轻松透出Wrapped类型：

```swift
public enum Optional<Wrapped> : ExpressibleByNilLiteral {
    case none
    case some(Wrapped)
​
    public init(_ some: Wrapped)
    
    public func map<U>(_ transform: (Wrapped) throws -> U) rethrows -> U?
​
    public func flatMap<U>(_ transform: (Wrapped) throws -> U?) rethrows -> U?
    
    public var unsafelyUnwrapped: Wrapped { get }
}
```

### Java

Java提供了Object作为动态任意类型，还实现了"假"泛型，就是在使用泛型的地方，编译器插入类型转换代码。将类型转换为原类型\(raw type\)，如果类型有特化，就换成特化类型，没有就替换成Object。这也叫做类型擦除\(type-erased\)，等代码经过编程成为字节码的时候，就没了原类型的信息。

有意思的是虽然有类型擦除，Java还是可以通过反射得到泛型的信息，但没法得到具体的原类型。除此之外，Java使用通配符?实现特化。

### Objective-C

Objective-C提供了id作为动态任意类型，几乎完全放弃了泛型，就提供了轻量泛型以供编译器检查。一个比较有意思的Objective-C泛型玩法，[在OC中实现tuple](https://github.com/WilliamZang/ZTuple)。

## 协变\(covariance\)、逆变\(contravariance\)、不变\(invariance\)

一些语言中的泛型支持协变、逆变。

* 协变\(covariance\)：接受比原类型更加具体的类型。
* 逆变\(contravariance\)：接受比原类型更加不具体的类型。
* 不变\(invariance\)：只接受原类型。

而为了保证安全性，协变和逆变是有条件的，那就是受里氏替换原则的约束。只能形成具体类型替换抽象类型，而不能反过来。这就造成了函数只能是返回值协变、参数逆变。

## 引用

[http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines.html\#S-templates](http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines.html#S-templates)

[https://github.com/wuye9036/CppTemplateTutorial](https://github.com/wuye9036/CppTemplateTutorial)

[https://github.com/apple/swift/blob/master/docs/Generics.rst](https://github.com/apple/swift/blob/master/docs/Generics.rst)

[https://docs.microsoft.com/zh-cn/dotnet/standard/generics/covariance-and-contravarianc](https://docs.microsoft.com/zh-cn/dotnet/standard/generics/covariance-and-contravariance)

