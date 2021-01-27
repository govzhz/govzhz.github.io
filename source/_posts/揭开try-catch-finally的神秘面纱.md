---
title: 揭开try-catch-finally的神秘面纱
date: 2020-05-17 22:53:17
toc: true
tags:
 - Java
 - JVM
categories: Java基础
---

根据 [JDK Tutorial](https://docs.oracle.com/javase/tutorial/essential/exceptions/finally.html) 的描述，除非在执行 try 或 catch 代码时线程被中断或 JVM 退出，finally 中的逻辑始终会执行。因此 finally 关键字常被用于释放资源，防止程序出现异常时出现资源泄露。本文主要探讨其在 JVM 层面的实现原理，以及 synchronized 关键字在类似场景的处理手段。首先来看一段简单的 try-finally 代码

```java
public void testWithTryFinally() {
  try {
    System.out.println("try");
  } finally {
    System.out.println("finally");
  }
}
```

<!-- more -->

其对应字节码如下

```
stack=2, locals=2, args_size=1
    0: getstatic      #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
    3: ldc            #6                  // String try
    5: invokevirtual  #7                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
    8: getstatic      #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
    11: ldc           #8                  // String finally
    13: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
    16: goto          30
    19: astore_1
    20: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
    23: ldc           #8                  // String finally
    25: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
    28: aload_1
    29: athrow
    30: return
Exception table:
    from    to  target type
       0     8    19   any
```

其中：

- 0 - 8 行是 try 中的语句
- 8 - 13，20 - 25 行是 finally 中的语句

那么为什么 finally 语句会出现两遍呢？其实这两次分别对应程序正常执行和异常执行的情况，8 - 13 行是在正常执行时会执行的 finally 语句，执行完成后通过 16 行的 `goto` 指令跳转到 `return` 指令返回；而 20 - 25 行则是由异常表（Exception table）进行触发，可以看到异常表会捕捉 0 - 8 行（不包含第8行）的字节码出现的任意异常，并且跳转至 19 行开始执行 finally 语句，最后通过 29 行 `athrow` 指令向上抛出异常。

那么如果增加 catch 呢，会有什么区别吗？

```java
public void testWithTryCatchFinally() {
    try {
      System.out.println("try");
    } catch (Exception e) {
      System.out.println("catch");
    } finally {
      System.out.println("finally");
    }
  }
```

其字节码如下：

```
stack=2, locals=3, args_size=1
    0: getstatic      #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
    3: ldc            #6                  // String try
    5: invokevirtual  #7                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
    8: getstatic      #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
    11: ldc           #8                  // String finally
    13: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
    16: goto          50
    19: astore_1
    20: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
    23: ldc           #10                 // String catch
    25: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
    28: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
    31: ldc           #8                  // String finally
    33: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
    36: goto          50
    39: astore_2
    40: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
    43: ldc           #8                  // String finally
    45: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
    48: aload_2
    49: athrow
    50: return
    Exception table:
        from    to  target type
            0     8    19   Class java/lang/Exception
            0     8    39   any
            19    28   39   any
```

- 0 - 5 行是 try 中的语句
- 20 - 25 行是 catch 中的语句
- 8 - 13,   28 - 33, 40 - 45 行是 finally 中的语句

大体上和之前逻辑相同，只不过这里 finally 中的语句又赋予了 catch 一遍，所以被 catch 后也能执行 finally 语句。根据上面两个例子，我们也验证了 [JDK Tutorial](https://docs.oracle.com/javase/tutorial/essential/exceptions/finally.html) 的描述，即**除非在执行 try 或 catch 代码时线程被中断或 JVM 退出，finally 中的逻辑始终会执行。**

# 包含控制语句的 try-finally

不知道大家有没有注意到，finally 语句的字节码前后总会出现 `astore`/`aload` 这样成对的指令，它们的作用是什么呢？根据指令本身的含义，我们可以知道

- `astore` 是将操作数栈顶存储到局部变量表
- `aload` 是将局部变量加载到操作数栈，以便后续 `ireturn` 指令将栈顶的值返回给方法调用者

它在我们分析包含控制转移语句（比如 return）的 try-catch-finally 有着至关重要的作用。举一个包含控制语句的 try-finally 的例子

```java
public int testWithTryReturnFinally() {
  int i = 0;
  try {
    return i;
  } finally {
    i++;
  }
}

# output: 0
```

其字节码如下：

```
stack=1, locals=4, args_size=1
    0: iconst_0
    1: istore_1
    2: iload_1
    3: istore_2
    4: iinc           1, 1
    7: iload_2
    8: ireturn
    9: astore_3
    10: iinc          1, 1
    13: aload_3
    14: athrow
    Exception table:
        from    to  target type
            2     4     9   any
```

- 0：将常量 0 加载到操作数栈
- 1：将栈顶 int 数值存入第 2 局部变量（其中由于这里是方法调用，第 1 局部变量被调用方执行 `invokevirtual` 指令将对象的引用隐式的传进来了），此时第 2 局部变量的值为 0
- 2：将第 2 局部变量（0）的值加载到操作数栈
- 3：**将栈顶的值存入第 3 局部变量（这里开始了 finally 的逻辑），这里相当于对 try 中的结果做了一次备份**，此时第 3 局部变量的值为 0
- 4：将第 2 局部变量（0）的值加 1
- 7：**加载第 3 局部变量的值到操作数栈（finally 语句结束），这里取出的是之前备份的值**
- 8：返回栈顶元素，此时由于栈顶是执行 finally 前备份的值，所以值为 0 
- 9：将栈顶的异常对象存入第 4 局部变量（进入到该阶段的指令一般由异常表触发，所以此时栈顶是异常对象）
- 10：将第 2 局部变量（0）的值加 1
- 13：将第 4 局部变量（异常对象）加载到操作数栈
- 14：将栈顶（异常对象）抛出

上述解释将对于最终返回结果比较重要的 3 和 7 指令进行了加粗，它展示了**虽然 finally 语句会执行，但是它的计算结果不一定会影响到返回值**，所以这里是容易被误解的地方，平常要避免这样使用。

那么是不是说放在 finally 中的计算都不会影响到 return 的结果呢？那当然不是，比如 return 不在 try 中，那么自然是会影响的，而如果 return 如果在 finally 中，又是什么样一种结果呢？比如：

```java
public int testWithTryReturnFinally() {
  int i = 0;
  try {
    return i;
  } finally {
    i++;
    return i;
  }
}

# output: 1
```

其字节码如下：

```
stack=1, locals=4, args_size=1
    0: iconst_0
    1: istore_1
    2: iload_1
    3: istore_2
    4: iinc           1, 1
    7: iload_1
    8: ireturn
    9: astore_3
    10: iinc          1, 1
    13: iload_1
    14: ireturn
    Exception table:
        from    to  target type
            2     4     9   any
```

看 7 和 13 指令，这里和刚才的有比较大的区别，它们用于返回的 iload_1 是原本的值（非备份），因此返回的结果是 1。当然还有一个更重要的区别是：**在 finally 中使用了 return 后丢失了 `athrow`，这意味着 try 中抛出的异常会丢失**（finally 中抛出的异常仍然会继续抛出），这是一个比较严重的问题。这里使用一个例子演示一下：

```java
public int testWithTryReturnFinallyException() {
  int i = 0;
  try {
    if(true) {
      throw new RuntimeException();
    }
    return i;
  } finally {
    i++;
    return i;
  }
}

# output: 1
```

上述代码并不会抛出异常，而是返回 1。所以 finally 中要避免使用 return，否则会得到意想不到的结果。经过上述几个例子，现在对 try-catch-finally 做了一个总结：

1. 除非在执行 try 或 catch 代码时线程被中断或 JVM 退出，否则 finally 中的逻辑始终会执行
2. finally 语句块会在 try block 的控制转移语句（如 return）之前执行，但不会影响最终返回的结果，除非 finally 抛出了异常或使用了 return 等控制转移语句
3. 避免在 finally 中使用 return，这会导致 try block 中的异常被丢失

# synchronized 如何保证始终执行 monitorexit

我们都知道 synchronized 对于同步语句块会使用 `monitorenter` 和 `monitorexit` 字节码指令，那么它们如何保证退出时始终执行 `monitorexit` 呢？答案其实和 finally 类似，我们举个例子：

```java
public void testWithSync() {
  synchronized (this) {
    System.out.println("sync");
  }
}
```

其字节码如下：

```
stack=2, locals=3, args_size=1
    0: aload_0
    1: dup
    2: astore_1
    3: monitorenter
    4: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
    7: ldc           #6                  // String sync
    9: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
    12: aload_1
    13: monitorexit
    14: goto          22
    17: astore_2
    18: aload_1
    19: monitorexit
    20: aload_2
    21: athrow
    22: return
    Exception table:
        from    to  target type
            4    14    17   any
            17   20    17   any
```

根据异常表可知即使同步块内容出现异常（4 - 14），仍然会跳转至 17 完成 `monitorexit` 的执行，这不就是 finally 的执行过程吗。但是这里相对 finally 有一个特殊的地方就是异常表对 17 - 20 行出现异常的情况进行了无限循环，而 17 - 20 行实际上就是执行 `monitorexit` 的过程，也就是说一旦 `monitorexit` 抛出异常，那么线程就会进入无限循环。根据 [The Java Virtual Machine Instruction Set](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.monitorexit) 介绍，`monitorexit` 会存在抛出 NullPointerException 和 IllegalMonitorStateException 两种异常，这就证明确实会存在无限循环的可能性。

> If *objectref* is `null`, *monitorexit* throws a `NullPointerException`.
>
> Otherwise, if the thread that executes *monitorexit* is not the owner of the monitor associated with the instance referenced by *objectref*, *monitorexit* throws an `IllegalMonitorStateException`.
>
> Otherwise, if the Java Virtual Machine implementation enforces the rules on structured locking described in [§2.11.10](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.11.10) and if the second of those rules is violated by the execution of this *monitorexit* instruction, then *monitorexit* throws an `IllegalMonitorStateException`.

那么这合理吗？在 2002 年就有一个 Bug 描述 [JDK-4414101 : synchronized statement generates catch around the monitorexit](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=4414101) 被提交，但最终被标记为非 bug。回答者认为无限循环是一个正确的行为，因为同步代码块的退出始终需要伴随着 monitor 的释放，一旦做不到这一点那么将线程放入无限循环中比执行其他操作更正确

