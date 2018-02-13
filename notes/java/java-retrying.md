# java-retrying: 简单灵活可配的java重试模块

## 由来

之前做项目的时候遇到大量这样的场景: 自个儿的服务通过http请求底层系统申请资源, 然后不断的每隔一段时间查询底层系统, 直到资源准备完成或超时.

这不同于一般的失败重试机制, 可以在一个线程里重试多次, 这样非常低效. 所以想到了异步重试, 起先自己通过spring的`TaskScheduler`, java8的`CompletableFuture`实现了一个非常简陋的异步重试类, 够用但是非常局限. 后来在网上搜到了[guava-retrying](https://github.com/rholder/guava-retrying), 用起来真的非常舒爽, 但是他只支持同步重试, 而且github上已经两三年没更新了, 看了他的代码非常简单, 于是打算自己在此基础上写一个, 就有了本文中的`java-retrying`.

> **Github地址**: https://github.com/lowzj/java-retrying

`java-retrying`完全去除了第三方依赖, 使用了jdk自身的`ScheduledExecutorService`, `CompletableFuture`实现异步重试.

比较一下`guava-retrying`和`java-retrying`:

名称 | JDK | 第三方依赖 | 同步重试 | 异步重试
---- | --- | ---- | -------- | --------
guava-retrying | 大于等于6 | guava,findbugs | Y | N
java-retrying | 大于等于8 | 无 | Y | Y

## 使用方法

使用起来非常简单灵活, 可以配置各种`WaitStategy`, `StopStategy`, 以及配置各种重试条件(根据结果, 异常类型等等).

举两个例子最直观:

* 同步重试
```java
Retryer<Integer> retryer = RetryerBuilder.<Integer>newBuilder()
    .withWaitStrategy(WaitStrategies.fixedWait(100L, TimeUnit.MILLISECONDS))
    .retryIfResult(num -> num != 5)
    .retryIfExceptionOfType(RuntimeException.class)
    .withStopStrategy(StopStrategies.stopAfterAttempt(7))
    .build();
try {
    retryer.call(noRuntimeExceptionAfter(4));
} catch (ExecutionException | RetryException e) {
    e.printStackTrace();
}
```
* 异步重试
```java
AsyncRetryer<Integer> asyncRetryer = RetryerBuilder.<Integer>newBuilder()
    .withWaitStrategy(WaitStrategies.fixedWait(100L, TimeUnit.MILLISECONDS))
    .retryIfResult(num -> num != 4)
    .retryIfExceptionOfType(RuntimeException.class)
    .withStopStrategy(StopStrategies.stopAfterAttempt(7))
    .withExecutor(ExecutorsUtil.scheduledExecutorService("example", 1))
    .buildAsyncRetryer();
CompletableFuture<Integer> future = asyncRetryer.call(noRuntimeExceptionAfter(3));
// get the result asynchronously
future.whenComplete((result, error) -> System.out.println(result));
// or get the result synchronously
try {
    System.out.println(future.get());
} catch (InterruptedException | ExecutionException e) {
    e.printStackTrace();
}
```

其中函数`noRuntimeExceptionAfter`如下:
```java
private Callable<Integer> noRuntimeExceptionAfter(final int attemptNumber) {
    return new Callable<Integer>() {
        private int count = 0;
        @Override
        public Integer call() throws Exception {
            if (count++ < attemptNumber) {
                throw new RuntimeException("count[" + (count - 1) + "] < attemptNumber[" + attemptNumber + "]");
            }
            return count;
        }
    };
}
```

异步重试时需要通过`RetryerBuilder#withExecutor`配置一个类型为`ScheduledExecutorService`的线程池, `java-retrying`中提供了一个工具类`ExecutorsUtil`可以非常方便的构造线程池. 如果不配置的话, 默认会提供一个名为`default-async-retry`, 核心线程数为5的线程池.

其他就没什么了, 更具体的可以看下代码, 代码本身也非常简单.

