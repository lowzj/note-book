# Lambda表达式的翻译原理

> 本篇文章翻译自: [Translation of Lambda Expressions](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-translation.html)
>
> 为什么要翻译此文:
> * 学习lambda表达
> * google了下，发现这篇翻译「[深入理解Java 8 Lambda](http://zh.lucida.me/blog/java-8-lambdas-insideout-language-features/)」非常好，其中提及了3篇「深入理解Java 8 Lambda」系列文章
>   * [State of Lambda](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-state-final.html)，译：[深入理解Java 8 Lambda 语言篇](http://zh.lucida.me/blog/java-8-lambdas-insideout-language-features/)
>   * [State of Lambda: Libraries Edition](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-libraries-final.html)，译：[深入理解Java 8 Lambda 类库篇](http://zh.lucida.me/blog/java-8-lambdas-inside-out-library-features/)
>   * [Translation of Lambda Expressions](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-translation.html)，没有找到翻译，所以决定试着翻译一下。

<pre align='center'>
正文
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

当编译器遇到lambda表达式的时候，它首先将lambda体转换为一个参数列表和返回类型与此lambda表达式匹配的方法，可能会有一些额外的参数(从词法域捕获的值，如果存在的话)。在lambda表达式被捕获处，编译器会生成一个`invokedynamic`调用点；当调用它时，返回一个lambda转换成的函数式接口实例。此调用点被称为给定lambda的lambda工厂(_lambda factory_)[^2]。Lambda factory的动态参数是从词法域捕获的值。Lambda facotry的引导方法(_bootstrap_ method, BSM)是Java语言运行库中的标准方法，被称之为lambda元工厂(_lambda metafactory_)。静态_bootstrap_参数在编译期捕获lambda的相关信息(lambda转换成的函数式接口、lambda体转换的方法句柄、有关此SAM[^3]类型是否可序列化的信息，等等)。

除了大多数方法引用不必被转换成一个新的方法外，方法引用以与lambda表达式相同的方式被处理；我们可以简单地加载被引用方法的方法句柄，然后传给元工厂(metafactory)。

## Lambda Body Desugaring

#### Desugaring example -- "stateless" lambdas

#### Desugaring example -- lambdas capturing immutable values

## The Lambda Metafactory

#### Lambda Capture

#### Static vs instance methods

#### Method reference capture

#### Varargs

#### Adaptations

#### Metafactory variants

## 序列化

## 其他方面

---

## Readings

* [InvokeDynamic指令](http://blog.csdn.net/zxhoo/article/details/38387141)

<pre align='center'>
脚注
</pre>

[^1]: 本文将`translate`翻译为`翻译`。
[^2]: 本文不再翻译`lambda factory`。
[^3]: SAM，全称Single Abstract Method。指仅有一个方法的接口，即函数式接口(functional interface)。
