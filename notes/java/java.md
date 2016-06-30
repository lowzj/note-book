# Java

## 利用反射机制获取方法的参数名称

举个栗子🌰
```java
import java.lang.reflect.Method;
import java.lang.reflect.Parameter;

public class ParameterNameTest {

    public void test(String name) {
    }

    public static void main(String[] args) throws NoSuchMethodException {
        Method method = ParameterNameTest.class.getDeclaredMethod("test", String.class);
        Parameter[] parameters = method.getParameters();
        for (Parameter parameter : parameters)
            System.out.println(parameter.getName());
    }
}
```
这个栗子里，获取到类中方法`test`，然后输出方法中的参数名称，期望的结果是**name**。
编译下，然后运行：
```sh
$ javac ParameterNameTest.java
$ java ParameterNameTest
arg0
```

debug一下`method.getParameters`的调用栈，看一下**arg0**是怎么来的：

```
4: java.lang.reflect.Executable#synthesizeAllParams
3: java.lang.reflect.Executable#getParameters0
2: java.lang.reflect.Executable#privateGetParameters
1: java.lang.reflect.Executable#getParameters
0: main
```

总结一下，获取方法中参数名称的基本过程如下：
* 首先从JVM中获取方法的所有参数: `getParameters0`
* 如果什么也没得到，那就合成参数: `synthesizeAllParams`
    * 参数在此方法中的位置，即是第几个参数
    * 参数名称生成规则: 'arg' + parameterIndex

OK, 看到这里，基本可以解释**arg0**是怎么来的了：因为JVM中没有能获得方法的参数，所以就直接合成了，**name**是第一个参数，所以合成后的参数名称是**arg0**。

那么问题来了，JVM如何才能得到方法的参数呢？了解一下`Parameter`类，构造函数上面有比较详细的注释：

```
 Package-private constructor for {@code Parameter}.
 
 If method parameter data is present in the classfile, then the
 JVM creates {@code Parameter} objects directly.  If it is
 absent, however, then {@code Executable} uses this constructor
 to synthesize them.
 
 @param name The name of the parameter.
 @param modifiers The modifier flags for the parameter.
 @param executable The executable which defines this parameter.
 @param index The index of the parameter.
```

注意到这一句

> If method parameter data is present in the **classfile**, then the
> JVM creates {@code Parameter} objects directly.

如果方法的参数数据在 **classfile** 中的话，那么JVM就会直接创建`Parameter`对象。**classfile** 是编译期生成的，所以呢，能不能取得参数的原始名称，在编译期就已经被决定了。
看一下`javac`的命令：

```sh
$ javac -help
...
  -parameters                Generate metadata for reflection on method parameters
...
```

好的吧，重新再编译一下执行：
```sh
$ javac -parameters ParameterNameTest.java
$ java ParameterNameTest
name
```

正确！

最后对比一下前后生成的 **classfile** 有什么不一样的地方：
```
# javac ParameterNameTest.java, classfile中 test 方法签名: 
test(Ljava/lang/String;)V

#javac -parameters ParameterNameTest.java, classfile中 test 方法签名:
test(Ljava/lang/String;)V MethodParameters name
```

后面多了一串`MethodParameters name`。

至此，利用反射来获取方法的参数名称基本搞明白了。但是依赖编译选项`-parameters`，必须打开此开关才能获取到代码中原始的参数名称。所以还是谨慎使用。

至于JVM是如何从**classfile**创建`Parameter`对象的，现在还没有搞明白。


### Readings

* [Obtaining Names of Method Parameters](https://docs.oracle.com/javase/tutorial/reflect/member/methodparameterreflection.html)
* [Java VM Specification - Chapter 4. The class File Format](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.7.24)
* [提炼 Java Reflection](http://lsy.iteye.com/blog/220264)
