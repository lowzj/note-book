# Lambda表达式的编译原理

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
本文概述了将Lambda表达式和方法引用从Java源代码编译(translate)[^1]成字节码的策略。Java的Lambda表达式由[JSR 335](http://jcp.org/en/jsr/detail?id=335)规范化并在OpenJDK [Lambda Project](http://openjdk.java.net/projects/lambda/)中实现。关于这个语言特性的概述可以在[State of the Lambda](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-state-final.html)中找到。

本文涉及当编译器遇到lambda表达式的时候，我们是如何得到它应当生成的字节码，还有语言运行时是如何参与到lambda表达式的评估中。本文但部分内容涉及到函数式接口的转换机制。

函数式接口是Java中labmbda表达式的核心方面。一个函数式接口是只有一个非Object类方法(non-Object method)的接口类，比如`Runnable`，`Comparator`，等等。（Java类库已经使用这些接口去表示回调多年了。）

Lambda表达式可以出现在被赋值给一个类型为函数式接口的变量的地方。比如：
```java
Runnable r = () -> { System.out.println("hello"); };
```
或者
```java
Collections.sort(strings, (String a, String b) -> -(a.compareTo(b)));
```

编译器生成的用于捕获这些lambda表达式的代码，取决于lambda表达式本身以及它所赋值的函数式接口类型。

#### Dependencies and notation

## 编译策略


## Lambda Body Desugaring

## The Lambda Metafactory

## 序列化

## 其他方面

---

<pre align='center'>
脚注
</pre>

[^1]: 本文将`translate`翻译为`编译`，诸如其他`翻译`、`转换`、`转译`，都觉得有点别扭，而且这一步确实也是Java编译器的工作。

