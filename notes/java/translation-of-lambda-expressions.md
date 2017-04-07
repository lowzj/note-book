# Lambda表达式: 翻译(Translation of Lambda Expressions)

> 本篇文章翻译自: [Translation of Lambda Expressions](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-translation.html)
>
> 为什么要翻译此文:
> * 学习lambda表达
> * google了下，发现这篇翻译「[深入理解Java 8 Lambda](http://zh.lucida.me/blog/java-8-lambdas-insideout-language-features/)」非常好，其中提及了3篇「深入理解Java 8 Lambda」系列文章
>   * [State of Lambda](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-state-final.html)，译：[深入理解Java 8 Lambda 语言篇](http://zh.lucida.me/blog/java-8-lambdas-insideout-language-features/)
>   * [State of Lambda: Libraries Edition](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-libraries-final.html)，译：[深入理解Java 8 Lambda 类库篇](http://zh.lucida.me/blog/java-8-lambdas-inside-out-library-features/)
>   * [Translation of Lambda Expressions](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-translation.html)，没有找到翻译，所以决定试着翻译一下。

## 关于本文档
本文档概述了将lambda表达式和方法引用从java源代码转译成字节码的策略。

