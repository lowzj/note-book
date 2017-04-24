# Lambda表达式的翻译原理

> 本篇文章翻译自: [Translation of Lambda Expressions](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-translation.html)
>
> 为什么要翻译此文:
> * 学习lambda表达
> * google了下，发现这篇翻译「[深入理解Java 8 Lambda](http://zh.lucida.me/blog/java-8-lambdas-insideout-language-features/)」非常好，其中提及了3篇「深入理解Java 8 Lambda」系列文章
>   * [State of Lambda](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-state-final.html)，译：[深入理解Java 8 Lambda 语言篇](http://zh.lucida.me/blog/java-8-lambdas-insideout-language-features/)
>   * [State of Lambda: Libraries Edition](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-libraries-final.html)，译：[深入理解Java 8 Lambda 类库篇](http://zh.lucida.me/blog/java-8-lambdas-inside-out-library-features/)
>   * [Translation of Lambda Expressions](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-translation.html)，没有找到翻译，所以决定试着翻译一下。

---

<pre align='center'>
译文始
</pre>

# Translation of Lambda Expressions

<p align='right'>
by <a href="http://www.oracle.com/us/technologies/java/briangoetzchief-188795.html">Brian Goetz</a>
</p>

## 关于本文
本文概述了将Lambda表达式和方法引用从Java源代码翻译(translate)[^1]成字节码的策略。Java的Lambda表达式由[JSR 335](http://jcp.org/en/jsr/detail?id=335)规范化并在OpenJDK [Lambda Project](http://openjdk.java.net/projects/lambda/)中实现。关于这个语言特性的概述可以在[State of the Lambda](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-state-final.html)中找到。

本文涉及当编译器遇到lambda表达式的时候，我们如何得到它应当生成的字节码，以及语言运行时是如何参与到lambda表达式的执行过程中的。本文大部分内容涉及到函数式接口的转换机制。

函数式接口是Java中lambda表达式的核心方面。函数式接口是有且只有一个非Object类方法(non-Object method)的接口类，比如`Runnable`，`Comparator`，等等。（Java类库已经使用这些接口去表示回调多年了。）

Lambda表达式仅可出现在被赋值变量的类型为函数式接口的地方。比如：
```java
Runnable r = () -> { System.out.println("hello"); };
```
或者
```java
Collections.sort(strings, (String a, String b) -> -(a.compareTo(b)));
```

编译器生成的用于捕获这些lambda表达式的代码，取决于lambda表达式本身以及它所赋值的函数式接口类型。

#### 依赖与符号
翻译方案依赖于[JSR 292](http://jcp.org/en/jsr/detail?id=292)中的几个特性，包括invokedynamic，方法句柄(method handle)，以及用于方法句柄和方法类型(method type)的增强LDC字节码形式。由于在Java源码中未能表示这些，所以我们的例子中将使用伪代码来表示这些特性。

* 对于方法句柄常量(method handle constants)：**MH([refKind] class-name.method-name)**
* 对于方法类型常量(method type constants)：**MT(method-signature)**
* 对于invokedynamic：**INDY((bootstrap, static args...)(dynamic args...))**

读者应该具备一些关于[JSR 292](http://jcp.org/en/jsr/detail?id=292)中的特性的基础知识。


翻译方案还假定一个新特性：用于使用常量方法句柄的反射API，这个特性正在由Java SE 8的JSR-292的专家组制定规范中。

## 翻译策略
我们有许多用字节码表示lambda表达式的方法，比如：内部类(inner class)、方法句柄(method handle)、动态代理，以及其他种种。其中每种途径都有支持者和反对者。在选择策略时，有两个相互矛盾的目标：最大化灵活性以使得在将来优化时不用提交一个特定的策略；与之相对的是保证类文件表示的稳定性。我们可以通过使用[JSR 292](http://jcp.org/en/jsr/detail?id=292)中的`invokedynamic`特性，把在字节码中创建lambda的二进制表示与运行时执行labmda表达式的机制分开，从而来实现这两个目标。我们为构造lambda表达式提供了一种方式(recipe)，并把真正的构造逻辑委托给语言运行时，而不是用生成创建对象的字节码来实现lambda表达式(比如调用内部类的构造方法)。此种方式(recipe)被编码到`invokedynamic`的静态和动态参数列表中。

使用`invokedynamic`使得我们将翻译策略的选择推迟到运行时。运行时实现可以动态地选择策略来执行lambda表达式。构造lambda的运行时实现选择被隐藏到标准化(即平台规范的一部分)API之后，因此静态编译器可以调用该API，并且各种JRE实现可以选择他们倾向的实现策略。`invokedynamic`机制允许这样做，并且没有后期绑定方法可能带来的性能损耗。

当编译器遇到lambda表达式的时候，它首先将lambda体转换为一个参数列表和返回类型与此lambda表达式匹配的方法，可能会有一些额外的参数(从词法作用域捕获的值，如果存在的话)。在lambda表达式被捕获处，编译器会生成一个`invokedynamic`调用点；当调用它时，返回一个lambda转换成的函数式接口实例。此调用点被称为给定lambda的lambda工厂(_lambda factory_)[^2]。Lambda factory的动态参数是从词法作用域捕获的值。Lambda facotry的引导方法(_bootstrap_ method, BSM)是Java语言运行库中的标准方法，被称之为lambda元工厂(_lambda metafactory_)。静态_bootstrap_参数在编译期捕获lambda的相关信息(lambda转换成的函数式接口、lambda体转换的方法句柄、有关此SAM[^3]类型是否可序列化的信息，等等)。

除了大多数方法引用不必被转换成一个新的方法外，方法引用以与lambda表达式相同的方式被处理；我们可以简单地加载被引用方法的方法句柄，然后传给元工厂(metafactory)。

## 将lambda体去糖[^4]

将lambda翻译为字节码的第一步就是将lambda体去糖成一个方法。

围绕着lambda体的去糖过程，要作出下面几个选择：

* 转化为静态方法还是实例方法？
* 去糖后的方法应该在什么类中？
* 去糖后的方法应该具有什么可见性？
* 去糖后的方法名是什么？
* 如果需要一个适配过程去解决lambda体签名和函数式接口方法前面之间的差异(比如装箱、拆箱、基本类型的宽型和窄型转换、可变参数转换，等等)，那么去糖后方法采用的签名是lambda体的，还是函数式接口方法的，或者介于这两者之间？另外谁来负责这个适配过程？
* 如果lambda从外部作用域(enclosing scop)中捕获参数，那么在去糖后方法的签名中该如何表示他们？(可以把他们当成相互独立的参数添加到参数列表的最前面或者最后，或者编译器可以把他们集中到一个结构参数中)。

一个将lambda体去糖的相关问题是，方法引用是否需要一个生成的适配器或者“桥接“方法。

编译器会为lambda表达式推断一个方法签名，包括参数类型、返回类型以及抛出的异常；我们称之为`自然签名`(_nature signature_)。Lambda表达式还拥有一个是函数式接口的目标类型(target type)；我们将`lambda描述符`(_lambda descriptor_)称为目标类型擦除的描述符的方法签名。_Lambda factory_的返回值，被称为lambda对象(_lambda object_)，实现了对应的函数式接口并捕获了lambda的行为。

同等条件下，去糖策略如下：

* 私有方法优于非私有方法。
* 静态方法优于实例方法。
* 最好是在lambda表达式出现的最内层类中将lambda体去糖。
* 签名应该匹配lambda体签名。
* 对于捕获的值，这些额外的参数应该放到参数列表的前面。
* 绝不转换方法引用。

但是，仍然有一些可能使我们偏离这个基准策略的异常用例。

#### 去糖例子 -- “无状态”lambda

Lambda表达式的一种最简单的形式，就是从外部作用域里捕获不到状态(无状态lambda，stateless lambda)：

```java
class A {
    public void foo() {
        List<String> list = ...
            list.forEach( s -> { System.out.println(s); } );
    }
}
```

此lambda的自然签名是`(String)V`；`forEach`方法有一个参数`Block<String>`，其lambda描述符为`(Ojbect)V`。编译器将lambda体去糖为一个签名为其自然签名的静态方法，同时为去糖后的函数体生成一个方法名。

```java
class A {
    public void foo() {
        List<String> list = ...
            list.forEach( [lambda for lambda$1 as Block] );
    }

    static void lambda$1(String s) {
        System.out.println(s);
    }
}
```

#### 去糖例子 -- 带不可变值的lambda

Lambda表达式的其他形式需要捕获外部_final_或_effectively fianl_[^5]的局部变量、外部实例中的字段(相当于捕获_final_型的外部`this`引用)。

```java
class B {
    public void foo() {
        List<Person> list = ...;
        final int bottom = ..., top = ...;
        list.removeIf( p -> (p.size >= bottom && p.size <= top) );
    }
}
```

上例中的lambda从外部作用域中捕获了_final_的局部变量`bottom`和`top`。

去糖后的方法签名是自然签名`(Person)Z`和一些追到到参数列表前面的额外参数。编译器拥有一定的权限决定如何去表示这些额外参数；比如单独地向前追加、装箱到一个类中、装箱到一个数组中，等等。最简单的方式是将它们单独地向前追加到参数列表里：

```java
class B {
    public void foo() {
        List<Person> list = ...;
        final int bottom = ..., top = ...;
        list.removeIf( [ lambda for lambda$1 as Predicate capturing (bottom, top) ]);
    }

    static boolean lambda$1(int bottom, int top, Person p) {
        return (p.size >= bottom && p.size <= top);
    }
}
```

可替代的选项是，捕获的值可以被装箱到一个结构(frame)或者数组中；关键点是，在去糖后lambda表达式的签名中的额外参数类型，与他们在_lambda factory_(动态)参数中的类型，是一致的。因为编译器对这两者均有控制权，并且在同一时间生成它们，所以编译器在捕获和包装上具有一定的灵活性。

## The Lambda Metafactory

#### Lambda Capture

#### Static vs instance methods

#### Method reference capture

#### Varargs

#### Adaptations

#### Metafactory variants

## Serialization

#### Accessibility

#### Class caching

#### Performance impact

## 其他方面

#### 桥接方法

函数式接口实际上可能不止一个非类方法，因为它可能存在桥接方法(bridge method)。

例如，在下面这个函数式接口类型`B`中:

```java
interface A<T> { void m(T t); }

interface B extends A<String> { void m(String s); }
```

接口`B`的基本方法(primary method)是`m(String)`，但同时它还有一个`m(String)`的桥接方法`m(Object)`。如若不然，当你将`B`转为`A`并调用`m`方法时，结果就会失败。

当我们将lambda表达式转为实现了函数式接口(比如`B`)的对象时，我们必须确保所有桥接方法连接正确，和基本方法一样有恰当的参数和返回类型适配器(类型转换)。通过生成特意的字节码或者分离编译组件，也可以在编译期找到那些不存在于函数式接口中的“额外”方法。我们可以采取`MethodHandleProxy`提供的方式，仅去桥接那些和基本方法有相同名称和相同参数数量的方法，而不是执行完整的JLS(Java语言规范)中的桥接算法。(如果此两者中任何与基本方法不兼容，在调用时会抛出`ClassCastException`，仅比在链接时抛出的错误些微少些信息量。)我们可以让编译器在元工厂(metafactory)中包含一个已知的在编译期有效的(known-valid-at-compile-time)桥接签名列表，但是这个很小的益处会增大类文件大小。

#### toString

通常情况下，lambda对象的`toString`方法继承自`Object`。但是，对于方法引用的共有非合成方法，我们可能会希望根据此方法实现中的类名或方法名来实现`toString`。例如，将`String::size`转换为`IntFn`时，我们可能需要`toString`返回字符串：**String::size()**、**java.lang.String::size()**、**String::size() as IntFn**，等等。

**TODO**: 如果我们支持命名式lambda(_named lambda_)的概念，当需要将名称以某种方式传递给`metafactory`时，我们可能会希望`toString`方法能够根据名称得出返回结果。

<pre align='center'>
译文终
</pre>

---

## Readings

* [InvokeDynamic指令](http://blog.csdn.net/zxhoo/article/details/38387141)
* [Java范型：类型擦除](https://segmentfault.com/a/1190000003831229)
* [官方文档：类型擦除](https://docs.oracle.com/javase/tutorial/java/generics/erasure.html)
* [桥接方法](http://blog.csdn.net/mhmyqn/article/details/47342577)

<pre align='center'>
脚注
</pre>

[^1]: 本文将`translate`直译为`翻译`。
[^2]: 本文直接使用`lambda factory`。
[^3]: SAM，全称Single Abstract Method。指仅有一个方法的接口，即函数式接口(functional interface)。
[^4]: 原文_Lambda Body Desugaring_，这里将`Desugaring`直接译为`去糖`，可以参考这里: https://zh.wikipedia.org/wiki/语法糖
[^5]: _effectively final_，指没有_final_修饰的变量或参数，如果在初始化之后，其值就不会改变，就是_effectively fianl_。在JavaSE8之前局部类仅能访问声明为_final_的局部变量，JavaSE8后局部类就可访问外部语句块中的_final_或者_effectively final_的变量。[Accessing Members of an Enclosing Class](http://docs.oracle.com/javase/tutorial/java/javaOO/localclasses.html#accessing-members-of-an-enclosing-class)

