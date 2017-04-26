# Lambda表达式的翻译原理

_2017-04-25_

> 本篇文章翻译自: [Translation of Lambda Expressions](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-translation.html)
>
> 为什么要翻译此文:
> * 学习lambda表达
> * google了下，发现这篇翻译「[深入理解Java 8 Lambda](http://zh.lucida.me/blog/java-8-lambdas-insideout-language-features/)」非常好，其中提及了3篇「深入理解Java 8 Lambda」系列文章
>   * [State of Lambda](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-state-final.html)，译：[深入理解Java 8 Lambda 语言篇](http://zh.lucida.me/blog/java-8-lambdas-insideout-language-features/)
>   * [State of Lambda: Libraries Edition](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-libraries-final.html)，译：[深入理解Java 8 Lambda 类库篇](http://zh.lucida.me/blog/java-8-lambdas-inside-out-library-features/)
>   * [Translation of Lambda Expressions](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-translation.html)，没有找到翻译，所以决定试着翻译一下。
>
> 欢迎指正。

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

本文涉及当编译器遇到lambda表达式的时候，我们如何得到它应当生成的字节码，以及语言运行时是如何参与到lambda表达式的计算过程中的。本文大部分内容涉及到函数式接口的转换机制。

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
我们有许多用字节码表示lambda表达式的方法，比如：内部类(inner class)、方法句柄(method handle)、动态代理，以及其他种种。其中每种途径都有支持者和反对者。在选择策略时，有两个相互矛盾的目标：最大化灵活性以使得在将来优化时不用提交一个特定的策略；与之相对的是保证类文件表示的稳定性。我们可以通过使用[JSR 292](http://jcp.org/en/jsr/detail?id=292)中的`invokedynamic`特性，把在字节码中创建lambda的二进制表示与运行时计算labmda表达式的机制分开，从而来实现这两个目标。我们为构造lambda表达式提供了一种方式(recipe)，并把真正的构造逻辑委托给语言运行时，而不是用生成创建对象的字节码来实现lambda表达式(比如调用内部类的构造方法)。此种方式(recipe)被编码到`invokedynamic`的静态和动态参数列表中。

使用`invokedynamic`使得我们将翻译策略的选择推迟到运行时。运行时实现可以动态地选择策略来计算lambda表达式。构造lambda的运行时实现选择被隐藏到标准化(即平台规范的一部分)API之后，因此静态编译器可以调用该API，并且各种JRE实现可以选择他们倾向的实现策略。`invokedynamic`机制允许这样做，并且没有后期绑定方法可能带来的性能损耗。

当编译器遇到lambda表达式的时候，它首先将lambda体(lambda body)转换为一个参数列表和返回类型与此lambda表达式匹配的方法，可能会有一些额外的参数(从词法作用域捕获的值，如果存在的话)。在lambda表达式被捕获处，编译器会生成一个`invokedynamic`调用点；当调用它时，返回一个lambda转换成的函数式接口实例。此调用点被称为给定lambda的lambda工厂(_lambda factory_)[^2]。Lambda factory的动态参数是从词法作用域捕获的值。Lambda facotry的引导方法(_bootstrap_ method, BSM)是Java语言运行库中的标准方法，被称之为lambda元工厂(_lambda metafactory_)。静态_bootstrap_参数在编译期捕获lambda的相关信息(lambda转换成的函数式接口、lambda体转换的方法句柄、有关此SAM[^3]类型是否可序列化的信息，等等)。

除了大多数方法引用不必被转换成一个新的方法外，方法引用以与lambda表达式相同的方式被处理；我们可以简单地加载被引用方法的方法句柄，然后传给元工厂(metafactory)。

## 将lambda体去糖[^4]

将lambda翻译为字节码的第一步就是将lambda体去糖成一个方法。

围绕着lambda体的去糖过程，要作出下面几个选择：

* 转化为静态方法还是实例方法？
* 去糖后的方法应该在什么类中？
* 去糖后的方法应该具有什么可见性？
* 去糖后的方法名是什么？
* 如果需要一个适配过程去解决lambda体签名和函数式接口方法前面之间的差异(比如装箱、拆箱、基本类型的宽型和窄型转换、变长参数转换，等等)，那么去糖后方法采用的签名是lambda体的，还是函数式接口方法的，或者介于这两者之间？另外谁来负责这个适配过程？
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
        List<String> list = ...;
        list.forEach( s -> { System.out.println(s); } );
    }
}
```

此lambda的自然签名是`(String)V`；`forEach`方法有一个参数`Block<String>`，其lambda描述符为`(Ojbect)V`。编译器将lambda体去糖为一个签名为其自然签名的静态方法，同时为去糖后的函数体生成一个方法名。

```java
class A {
    public void foo() {
        List<String> list = ...;
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

## Lambda元工厂

Lambda捕获由`invokedynamic`调用点实现，其静态参数描述了lambda体和lambda描述符的特征，其动态参数(如果有)即捕获值。当此调用点被调用时，它会返回一个lambda体和描述符相应的绑定了捕获值的lambda对象。此调用点的引导方法(bootstrap method)是一个规范化的平台方法，被称为**lambda元工厂**(_lambda metafactory_)。(我们可以让所有形式的lambda仅有一个单独的元工厂，或者对于普遍情况有专门的版本。) 虚拟机对于每个捕获点仅会调用一次元工厂；在此之后，虚拟机会链接此调用点并完成工作。因为调用点是被懒加载的，所以从未调用的工厂方法(factory site)永远不会被链接。基础的元工厂的静态参数列表如下所示：

```java
metaFactory(MethodHandles.Lookup caller, // provided by VM
        String invokedName,          // provided by VM
        MethodType invokedType,      // provided by VM
        MethodHandle descriptor,     // lambda descriptor
        MethodHandle impl)           // lambda body
```

* 头三个参数(`caller`，`invokedName`，`invokedType`)是由虚拟机在调用点链接期自动添加的。
* 参数`descriptor`标识了lambda转换成的函数式接口方法。(通过方法句柄的反射API，元工厂可以取得函数式接口类的名称，和其基本方法的方法签名的名称。)
* 参数`impl`标识了lambda方法，或是去糖后的lambda体，或是用方法引用命名的方法。

函数式接口的方法和其实现类的方法的方法签名，也可能有一些不同之处。实现类可能会有一些与捕获参数相应的额外参数，其余的参数也可能不完全匹配；**适配器**(Adaptation)小节介绍了某些允许的适配器(自类型，装箱)。

#### Lambda捕获

我们现在准备好介绍将lambda表达式和方法引用转换为函数式接口的翻译过程。我们可以将例子`A`翻译如下：

```java
class A {
    public void foo() {
        List<String> list = ...;
        list.forEach(indy((MH(metaFactory), MH(invokeVirtual Block.apply),
                        MH(invokeStatic A.lambda$1))()));
    }

    private static void lambda$1(String s) {
        System.out.println(s);
    }
}
```

因为`A`中的lambda是无状态的(stateless)，所以其lambda工厂方法的动态参数列表是空的。

对于例子`B`，其动态参数列表是非空的，因为我们必须将`bottom`和`top`提供给_lambda factory_：

```java
class B {
    public void foo() {
        List<Person> list = ...;
        final int bottom = ..., top = ...;
        list.removeIf(indy((MH(metaFactory), MH(invokeVirtual Predicate.apply),
                        MH(invokeStatic B.lambda$1))( bottom, top ))));
    }

    private static boolean lambda$1(int bottom, int top, Person p) {
        return (p.size >= bottom && p.size <= top);
    }
}
```

#### 静态方法与实例方法

Lambda表达式如上一节所说，可以被翻译为静态方法，因为其无论如何都不会使用外部对象实例(不引用`this`、`super`以及外部实例的成员)。总而言之，我们将使用`this`、`super`和捕获外部实例成员的lambda成为实例捕获型lambda(_instance-capturing lambda_)。

非实例捕获型lambda(_non-instance-capturing lambda_)被翻译为私有静态方法，实例捕获型lambda则被翻译为私有实例方法。这样简化了实例捕获型lambda的去糖逻辑，lambda体中的名称即意味着去糖后方法中的名称，并且非常契合现有的实现技术(方法句柄绑定)。当捕获一个实例捕获型lambda时，接收者(`this`)被作为第一个动态参数。

一个例子，考虑一个捕获了成员属性`minSize`的lambda：

```java
list.filter(e -> e.getSize() < minSize )
```

我们将这个lambda转化为一个实例方法，并把接收者当作第一个捕获参数：

```java
list.forEach(INDY((MH(metaFactory), MH(invokeVirtual Predicate.apply),
                MH(invokeVirtual B.lambda$1))( this ))));

private boolean lambda$1(Element e) {
    return e.getSize() < minSize;
}
```

因为lambda体被翻译为私有方法，所以当将行为方法句柄传递给元工厂时，捕获点应当加载一个常量方法句柄：对于实例方法，其(该常量方法句柄)引用种类(reference kind)为`REF_invokeSpecial`；对于静态方法，其引用种类为`REF_invokeStatic`。

我们之所以可以转化成私有方法，是因为私有方法对捕获类是可见的，因而私有方法的方法句柄可被元工厂调用。如果元工厂用生成字节码来实现目标函数式接口，而不是直接调用方法句柄，那它将会通过免除可见性检查的`Unsafe.defineClass`来加载这些类。


#### 方法引用捕获

方法引用(method reference)有多种形式，与lambda类似，可以分为实力型捕获和非实例捕获。

非实例捕获型方法引用包括：
* 静态方法引用(static method reference)，如`Integer::parseInt`，使用引用种类`invokeStatic`捕获。
* 未绑定实例方法引用(unbound instance method reference)，如`String::length`，使用引用种类`invokeVirtual`捕获。
* 顶层构造方法应用(top-level constructor reference)，如`Foo::new`，使用引用种类`invokeNewSpecial`捕获。

当捕获一个非实力型捕获型方法引用时，其捕获参数列表总为空，如：

```java
list.filter(String::isEmpty)
```

被翻译为：

```java
list.filter(indy((MH(metaFactory), MH(invokeVirtual Predicate.apply),
            MH(invokeVirtual String.isEmpty))()))
```

实例捕获型方法引用包括：
* 绑定方法引用(bound instance methed reference)，如`s::length`，使用引用种类`invokeVirtual`捕获。
* 父类方法引用(super method reference)，如`super::foo`，使用引用种类`invokeSpecial`捕获。
* 内部类构造方法引用(inner class constructor reference)，如`Inner::new`，使用引用种类`invokeNewSpecial`捕获。

当捕获一个实例捕获型方法引用时，捕获参数列表总有一个单一的参数：对于父类或内部类构造方法引用时是`this`，对于绑定实例方法引用则是指定的接收者。

#### 变长参数

如果一个变长参数方法的方法引用被转换成一个非变长参数方法的函数式接口，编译器必须生成桥接方法，并且捕获此桥接方法的方法句柄而不是目标方法的方法句柄。此桥接方法必须处理任何需要的参数类型的适配，以及从变长参数到非变长参数的转换。例如：

```java
interface IIS {
    void foo(Integer a1, Integer a2, String a3);
}

class Foo {
    static void m(Number a1, Object... rest) { ... }
}

class Bar {
    void bar() {
        SIS x = Foo::m;
    }
}
```

这里，编译器需要生成一个桥接方法去执行下面的适配逻辑：将第一个参数类型从`Number`适配到`Integer`，剩余的参数汇集到一个`Object`数组中。

```java
class Bar {
    void bar() {
        SIS x = indy((MH(metafactory), MH(invokeVirtual IIS.foo),
                    MH(invokeStatic m$bridge))( ));
    }

    static private void m$bridge(Integer a1, Integer a2, String a3) {
        Foo.m(a1, a2, a3);
    }
}
```

#### 适配

去糖后的lambda方法有参数列表和返回类型：`(A1..An) -> Ra`(如果去糖后的方法是一个实例方法，则接收者被作为第一个参数)。类似的，函数式接口方法也有参数列表和返回类型：`(F1..Fm) -> Rf`(无接收者这一参数)。工厂方法的动态参数列表有参数类型：`(D1..Dk)`。如果是实例捕获型lambda，则其第一个动态参数必须是此接收者。

这些参数长度之和如下：`k+m == n`。就是说，lambda体的参数列表的长度，应该和动态参数列表与函数式接口参数列表的长度之和一样。

我们将lambda体的参数列表`(A1..An)`划为`(D1..Dk H1..Hm)`，其中`D`对应“额外”(动态)参数，`H`对应函数式接口参数。

要求参数`Hi`适配`Fi`，`i`从`1`到`m`；类似的，要求返回值`Ra`适配`Rf`。对于类型`T`与类型`U`，若：
* `T` == `U`
* `T`是基本类型，`U`是引用类型，且通过装箱转换可以将`T`转换为`U`
* `T`是引用类型，`U`是基本类型，且通过拆箱转换可以将`T`转换为`U`
* `T`和`U`均是基本类型，且通过基本类型窄型转换可以将`T`转换为`U`
* `T`和`U`均是引用类型，且`T`可强制转换为`U`

则类型`T`适配于类型`U`。

适配由元工厂在链接期检查，在捕获期执行。

#### 元工厂变种

对所有形式的lambda都仅有一个单一的元工厂是实用的。但是，将元工厂划分为不止一种，似乎更好些：

* _快速路径_(_fast path_)[^6]版，支持非序列化lambda和非序列化的静态或非绑定实例方法引用。
* _序列化_版，支持所有种类的序列化lambda和方法引用。
* 如若必要，还有一个_厨房水槽_(_kitchen sink_)[^7]版，支持转换功能的任意组合。

厨房水槽版需要一个额外的标志参数`flags`来选择选项，可能也需要其他特定选项的参数。序列化版则可能需要一个额外的与序列化相关的参数。

由于元工厂不是由用户直接调用的，所以没有因为用多种方式做同一件事情而引起困惑。通过删除不必要的参数，使得类文件变得更小。快速路径选项减少了虚拟机内置lambda转换操作的障碍，使其可被视为装箱操作，同时使拆箱优化更容易。

## 序列化

我们的动态翻译策略规定了一个动态序列化策略。假如我们希望能够从今天为每个lambda编造一个内部类，明天则转换到使用动态代理，那么今天序列化的lambda对象在明天反序列化时必须转为动态代理。通过为lambda表达式定义中间的序列格式，并使用`readResolve`和`writeReplace`在lambda对象和序列化格式(serialized form)间进行转换，从而完成上面的要求。这个序列后的格式必须包含通过元工厂重建lambda对象所需的全部信息。举个例子，序列化格式可以如下：

```java
public interface SerializedLambda extends Serializable {
    // capture context
    String getCapturingClass();

    // SAM descriptor
    String getDescriptorClass();
    String getDescriptorName();
    String getDescriptorMethodType();

    // implementation
    int getImplReferenceKind();
    String getImplClass();
    String getImplName();
    String getImplMethodType();

    // dynamic args -- these will individually need to be Serializable too
    Object[] getCapturedArgs();
}
```

这里，`SerializedLambda`接口提供了存在于原始lambda捕获点的所有信息。当捕获一个可序列化的lambda时，元工厂必须返回一个实现了`writeReplace`方法的对象；`writeReplace`方法需返回一个具有`readResolve`方法的`SerializedLambda`的实现类；而`readResolve`方法负责重建lambda对象。

#### 可见性

一个挑战是，反序列化代码需要为实现类方法构造一个方法句柄。虽然序列化格式提供了如此做的所有信息 -- 种类(kind)、类(class)、名称(name)、类型(type) -- 由此可以使用`MethodHandles.Lookup`暴露的`findXxx`方法中的一个来构造方法句柄，但是lambda方法或者被引用的方法对于`SerializedLambda`的实现类是不可见的(要么由于方法不可见，要么由于外部类不可见)。

由于实现类的方法句柄是用该类的访问权限加载的，所以对于创造lambda的lambda工厂点(lambda factory site)来说，这并不是一个问题。但是，为了防止引入安全风险，我们希望最小化任何用在反序列化过程中的权限提升，当然更不想修改JVM的可见性规则。

一种可能是确保可序列化lambda和方法引用的实现类方法是公有类中的公有方法。这也许已经是真的(方法引用`String::length`)，或者可以很容易的实现(对于公有类，我们可以将lambda转换为公有方法)。但是这依然是不可取的，因为它暴露了内部成员为公有方法，这与我们翻译非序列化lambda不一致。在某些情况下，还需要一些有效的技巧，比如当lambda在一个非公有类中时，则创建一个公有的_边车_(_sidecar_)[^8]类。(在某种程度上，暴露为公有的是不可避免的，因为那就是序列化所做的事 -- 提供一个外部的公有的事实上的类的构造函数。但是我们希望尽量减少这种暴露。)

一个更好的方案是将反序列化操作委派回给做lambda捕获的类。一个关键的安全挑战是，不要让反序列化机制允许攻击者通过构造篡改的字节流来构造一个可以调用任意私有方法的lambda对象。序列化天然暴露(函数式接口、行为方法、捕获参数类型)的特定组合的**构造函数**；通过委派回给捕获类，可以在继续之前验证字节流表示的是一个合法有效的组合。一旦验证组合有效，就可以通过元工厂调用来构造lambda，使用其自己的访问权限来加载方法句柄。

为了做到这点，捕获类应当有一个可以从序列化层调用的辅助方法，其实就是`readObject`、`writeObject`、`readResolve`、`writeReplace`。我们将其称为`$deserialize$(SerializedLambda)`。序列化层唯一需要的提权操作就是调用此方法(很可能是私有的)。

当编译一个捕获来可序列化的lambda类时，编译器知道(函数式接口、行为方法、捕获参数类型)中的哪个组合该被捕获为可序列化lambda。`$deserialize$`应当仅支持这些反序列化组合。

考虑下面这个捕获类两个可序列化的lambda类：

```java
class Foo {
    void moo() {
        SerializableComparator<String> byLength = (a,b) -> a.length() - b.length();
        SerializablePredicate<String> isEmpty = String::isEmpty;
        ...
    }
}
```

我们可以翻译如下：

```java
class Foo {
    void moo() {
        SerializableComparator<String> byLength
            = indy(MH(serializableMetafactory), MH(invokeVirtual SerializableComparator.compare),
                    MH(invokeStatic lambda$1))());
        SerializablePredicate<String> isEmpty
            = indy(MH(serializableMetafactory), MH(invokeVirtual SerializablePredicate.apply),
                    MH(invokeVirtual String.isEmpty)());
        ...
    }

    private static int lambda$1(String a, String b) { return a.length() - b.length(); }

    private static $deserialize$(SerializableLambda lambda) {
        switch(lambda.getImplName()) {
            case "lambda$1":
                if (lambda.getSamClass().equals("com/foo/SerializableComparator")
                        && lambda.getSamMethodName().equals("compare")
                        && lambda.getSamMethodDesc().equals("...")
                        && lambda.getImpleReferenceKind() == REF_invokeStatic
                        && lambda.getImplClass().equals("com/foo/Foo")
                        && lambda.getImplDesc().equals(...)
                        && lambda.getInvocationDesc().equals(...))
                    return indy(MH(serializableMetafactory),
                            MH(invokeVirtual SerializableComparator.compare),
                            MH(invokeStatic lambda$1))(lambda.getCapturedArgs()));
                break;

            case "isEmpty":
                if (lambda.getSamClass().equals("com/foo/SerializablePredicate")
                        && lambda.getSamMethodName().equals("apply")
                        && lambda.getSamMethodDesc().equals("...")
                        && lambda.getImpleReferenceKind() == REF_invokeVirtual
                        && lambda.getImplClass().equals("java/lang/String")
                        && lambda.getImplDesc().equals(...)
                        && lambda.getInvocationDesc().equals(...))
                    return indy(MH(serializableMetafactory),
                            MH(invokeVirtual SerializablePredicate.apply),
                            MH(invokeVirtual String.isEmpty)(lambda.getCapturedArgs));

                break;
        }
        throw new ...;
    }
}
```

`$deserialize$`方法知道哪些lambda已被这个类捕获，所以可以根据此列表检查序列化格式，然后使用一个可以与捕获点一样共享相同引导索引(bootstrap index)的同一调用点来重新构造lambda。(或者，通过将捕获分解为一个私有方法来共享同一个实际捕获点，由此共享相同的链接状态(linkage state)；这样也许能简化下节**类缓存**提到的某些问题。)

如果一个恶意的调用者欺骗我们去反序列化一段恶意字节流，也仅适用于那些事实上确实是在该编译单元中lambda转换目标的方法，即我们在翻译为可序列化内部类时所暴露的方法。因为是与去糖后方法在同一个编译单元，所以不会在重新编译时引入额外的名称不稳定性问题。

这就允许了一个对所有lambda体都简单的、通用的(one-size-fits-all)去糖策略 -- 我们对序列化和非序列化lambda使用同样的策略。这个策略让所有去糖后lambda体都是私有的，不需要边车类或者可访问的桥接方法，且唯一需要提权的操作仅是调用`$deserialize$`。

为了减少过多暴露带来的类加载攻击(攻击者创建一个序列化lambda的描述，意图强制加载该类，以产生其静态初始化方法引起的副作用)，最好是将`SerializedLambda`接口完全处理为类名称的名义标识符(nominal identifier)，而不是`Class`对象。

#### 类缓存

在众多可能的翻译策略中，我们都需要生成新的类。例如，如果每个lambda我们都生成一个类(在运行时而不是编译期编造内部类)，我们只在给定的_lambda factory_第一次被调用时才生成此类。之后，对先前的那个_lambda factory_的调用都将重用第一次调用时生成的类。

对于可序列化的lambda，在类生成时可能会触发两点：捕获点，以及在`$deserialize$`代码中相应的工厂调用点。理想的情况是(尽管非必须)，不管哪条路径先触发，通过两者中任何一个生成的对象都应有相同的类，这要求每个lambda捕获有唯一的key，且缓存在此给定可序列化lambda的两个捕获点之间是共享的。

```java
class SerializationExperiment {
    interface Foo extends Serializable { int m(); }

    public static void main(String[] args) {
        Foo f1, f2;
        if (args[0].equals("r")) {
            // read file 'foo.ser' and deserialize into f1
        }

        f2 = () -> 3;

        if (args[0].equals("w")) {
            // serialize f2 and write into file 'foo.ser'
            // read file 'foo.ser' and deserialize into f1
        }

        assert f1.getClass() == f2.getClass();
    }
}
```

如果我们运行这个程序两次：

```sh
java -ea SerializationExperiment w
java -ea SerializationExperiment r
```

期望的是，不管类编造(class spinning)发生在第一次反序列化时还是第一次调用元工厂时，都会运行成功。

#### 性能影响

可序列化性给lambda增加了一些额外的性能损失，因为lambda对象需要携带足够多的状态信息来有效地重建元工厂的静态和动态参数列表。这可能意味着实现类中额外的字段，额外的构造函数初始化工作，以及对翻译策略的约束(例如，我们不能使用方法句柄代理，因为其生成对象不会实现必须的`writeReplace`方法)。因此，更好的方案是把可序列化的lambda分离出来，而不是让所有lambda都可序列化致使这些性能损失增加到所有lambda上。

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

通常情况下，lambda对象的`toString`方法继承自`Object`。但是，对于方法引用的公有非合成方法，我们可能会希望根据此方法实现中的类名或方法名来实现`toString`。例如，将`String::size`转换为`IntFn`时，我们可能需要`toString`返回字符串：**String::size()**、**java.lang.String::size()**、**String::size() as IntFn**，等等。

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
[^6]: _快速路径_(_fast path_)，指在一个程序中比起一般路径有更短的指令路径长的路径。有效的快速路径会在处理最长出现的情形时比一般路径更有效率。
[^7]: _厨房水槽_(_kitchen sink_)，用厨房水槽比喻，指杂七杂八，这里意思应该是，除了那两个版本中特定的情形外，此版本使用其他形式的lambda。
[^8]: _边车_(_sidecar_)，了解一下边车的构造，大概就能意会边车类(sidecar class)是干嘛的-_-!!。[WIKI:边车](https://zh.wikipedia.org/wiki/%E9%82%8A%E8%BB%8A)。

