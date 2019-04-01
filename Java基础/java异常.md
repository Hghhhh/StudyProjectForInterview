![](G:\Administrator\Pictures\java基础\Throwable.jpg)

**Throwable是Error和Exception的父类，用来定义所有可以作为异常被抛出来的类。**

**Error和Exception区分*：

**Error（错误）是程序无法处理的错误，表示运行应用程序中较严重问题。大多数错误与代码编写者执行的操作无关，而表示代码运行时 JVM（Java 虚拟机）出现的问题**。也就是说Error我们的应用程序没办法怎么做。

Exception则是可以被抛出的基本类型，我们需要主要关心的也是这个类。

**Exception又分为RunTimeException和其他Exception。**

RunTimeException和其他Exception区分：

其他Exception，受检查异常。可以理解为错误，**必须要开发者解决以后才能编译通过**，解决的方法有两种，1：throw到上层，2，try-catch处理。
RunTimeException：运行时异常，又称不受检查异常，不受检查！不受检查！！不受检查！！！重要的事情说三遍，因为不受检查，所以在代码中可能会有RunTimeException时**Java编译检查时不会告诉你有这个异常**，**但是在实际运行代码时则会暴露出来**，比如经典的1/0，空指针等。如果不处理也会被Java自己处理。

