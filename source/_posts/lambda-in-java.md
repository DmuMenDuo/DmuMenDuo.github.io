---
title: 一个lambda的表达式是如何在java中执行的
cover: 'http://pic.menduo.xyz/20190711005529.png'
categories: Java
tags: 
    - Java
    - lambda
    - 学习笔记
date: 2019-07-11 00:34:58
---

实习的过程中，发现项目中大量的使用了Lambda的写法来提高开发效率。一直很好奇它底层到底是如何实现的？为什么可以把一个函数当做一个参数来进行传递和执行？ 

<!-- more -->


## 1.一个lambda函数是如何被执行的：

### 1.1一个简单的demo源代码如下：

这是一段简单的测试代码，包含了一个`lambda`函数，其中`HelloWorldLambda`是我定义的一个函数式接口,在这个接口的里面只包含一个抽象方法方法`hello`。
```java
public class Main {
    public static void main(String[] args) {
         HelloWorldLambda helloWorldLambda = () -> System.out.println("hello menduo");
         helloWorldLambda.hello();
    }
}
```
### 1.2编译阶段：三个需要注意的点

1. `lambda`表达式被编译生成了静态内部方法`lambda$main$0`
 ![1](http://pic.menduo.xyz/20190711003954.png)
2. 调用`lambda`处，生成`lambda`的字节码执行指令 : `invokedynamic #2,0 // InvokeDynamic #0:hello:()LHelloWorldLambda;`。 
![2](http://pic.menduo.xyz/20190711004404.png)
3. 这里的`#0`指的是编译生成的`Boostrap Method`：后面的是需要用到的参数，也就是实现`lambda`的接口对应的方法
![3](http://pic.menduo.xyz/20190711004421.png)

### 1.3执行阶段：
1. `invokedynamic #2` ==> 执行`BoostrapMethods` 的 `0`。⬆️上图

2. 可以看到,他是一个 `invokestatic指令`。执行了一个静态方法`LambdaMetafactory.metafactory(...)`

3. `LambdaMetafactory.metafactory(...)`  会执行`buildCallSite()` 返回一个`CallSite`，`CallSite`是一个方法句柄。
![1.3.3](http://pic.menduo.xyz/20190711004538.png)
 
1. `buildCallSite()` 主要做的事情就是，1.用asm生成一个实现lambda函数接口的代理类;2.然后返回一个方法句柄，就是lambda生成的静态方法的代理方法的句柄。
```java
@Override
CallSite buildCallSite() throws LambdaConversionException {
    final Class<?> innerClass = spinInnerClass(); //1
    if (invokedType.parameterCount() == 0) {
        final Constructor<?>[] ctrs = //...略
        try {
            Object inst = ctrs[0].newInstance();
            return new ConstantCallSite(MethodHandles.constant(samBase, inst)); //2
        }
    } else {
        try {
            UNSAFE.ensureClassInitialized(innerClass);
            return new ConstantCallSite(
                    MethodHandles.Lookup.IMPL_LOOKUP
                            .findStatic(innerClass, NAME_FACTORY, invokedType)); //2
        }
    }
}
```
最终 invokeinterface 查找调用点的方法表，执行下面代理方法。
![](http://pic.menduo.xyz/20190711004742.png)

## 2.相关的知识点

### 2.1 方法句柄：MethodHandle

#### 2.1.1 MethodHandle作用

MethodHandle是一种拿到方法,并执行的手段，类似于反射机制拿到方法。

#### 2.1.2 MethodHandle和反射拿到方法之间的不同

1. 性能上考虑：`MethodHandle`比反射更迅速，访问检查是在创建时而不是在执行时进行的。（而且从下面的例子中看到，它操作的是字节码`findStatic`，而反射拿到方法的是在`java代码层`面上)

2. 灵活性考虑：反射要更加的灵活。`MethodHandle`没有列举类中成员，获取属性访问标志之类的机制。（就只有`findXXX`

#### 2.1.3 MethodHandle使用

1. 方法在哪里。methodHandle.Lookup指定的类
2. 方法名是什么。第二个参数
3. 方法的参数签名和返回类型。 MethodType

一个`methodhandle`的小例子: MethodType 的第一个参数是返回类型，后面的是方法签名参数上的类型
```java
public class MethodHandleMain {
    public static void main(String[] args) {
        MethodHandles.Lookup  lookup = MethodHandles.lookup();
        MethodType methodType = MethodType.methodType(Integer.class,Integer.class,Integer.class);
        try {
            MethodHandle methodHandle = lookup.findStatic(MethodHandleMain.class,"sum",methodType);
            Integer x = (Integer) methodHandle.invoke(1,2);
            System.out.println(x);
        } catch (Throwable e) {
            e.printStackTrace();
        }
    }
    private static Integer sum(Integer a, Integer b) {
        return a + b;
    }
}
```

### 2.2 字节码生成：ASM

#### 2.2.1 ASM框架：

ASM是一个字节码操作的框架。可以动态的生成和修改字节码文件

#### 2.2.2 用途：

1. lambda表达式的解析。（这里用到了
2. cglib动态代理类的生成。
3. jacoco插桩也会用到。

#### 2.3 虚拟机指令：invokedynamic, invokeinterface,invokestatic...


| 指令 |说明 |
| --- | --- |
|invokeinterface |用以调用接口方法，在运行时搜索一个实现了这个接口方法的对象，找出适合的方法进行调用。（Invoke interface method）|
|invokevirtual | 指令用于调用对象的实例方法，根据对象的实际类型进行分派（Invoke instance method; dispatch based on class）|
|invokestatic | 用以调用类方法（Invoke a class (static) method ）|
|invokespecial | 指令用于调用一些需要特殊处理的实例方法，包括实例初始化方法、私有方法和父类方法。（Invoke instance method; special handling for superclass, private, and instance initialization method invocations ）|
|invokedynamic | JDK1.7新加入的一个虚拟机指令，相比于之前的四条指令，他们的分派逻辑都是固化在JVM内部，而invokedynamic则用于处理新的方法分派：它允许应用级别的代码来确定执行哪一个方法调用，只有在调用要执行的时候，才会进行这种判断,从而达到动态语言的支持。(Invoke dynamic method) |


## 4.附录：打印完整的字节码文件 javap -p -v LambdaMain2.class
```java
Classfile /Users/menduo/IdeaProjects/menduodemo/target/classes/LambdaMain2.class
  Last modified 2019-7-9; size 1179 bytes
  MD5 checksum 7597c1d01af4829107e2659b508fbc77
  Compiled from "LambdaMain2.java"
public class LambdaMain2
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #8.#25         // java/lang/Object."<init>":()V
   #2 = InvokeDynamic      #0:#30         // #0:hello:()LHelloWorldLambda;
   #3 = InterfaceMethodref #31.#32        // HelloWorldLambda.hello:()V
   #4 = Fieldref           #33.#34        // java/lang/System.out:Ljava/io/PrintStream;
   #5 = String             #35            // hello menduo
   #6 = Methodref          #36.#37        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #7 = Class              #38            // LambdaMain2
   #8 = Class              #39            // java/lang/Object
   #9 = Utf8               <init>
  #10 = Utf8               ()V
  #11 = Utf8               Code
  #12 = Utf8               LineNumberTable
  #13 = Utf8               LocalVariableTable
  #14 = Utf8               this
  #15 = Utf8               LLambdaMain2;
  #16 = Utf8               main
  #17 = Utf8               ([Ljava/lang/String;)V
  #18 = Utf8               args
  #19 = Utf8               [Ljava/lang/String;
  #20 = Utf8               helloWorldLambda
  #21 = Utf8               LHelloWorldLambda;
  #22 = Utf8               lambda$main$0
  #23 = Utf8               SourceFile
  #24 = Utf8               LambdaMain2.java
  #25 = NameAndType        #9:#10         // "<init>":()V
  #26 = Utf8               BootstrapMethods
  #27 = MethodHandle       #6:#40         // invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #28 = MethodType         #10            //  ()V
  #29 = MethodHandle       #6:#41         // invokestatic LambdaMain2.lambda$main$0:()V
  #30 = NameAndType        #42:#43        // hello:()LHelloWorldLambda;
  #31 = Class              #44            // HelloWorldLambda
  #32 = NameAndType        #42:#10        // hello:()V
  #33 = Class              #45            // java/lang/System
  #34 = NameAndType        #46:#47        // out:Ljava/io/PrintStream;
  #35 = Utf8               hello menduo
  #36 = Class              #48            // java/io/PrintStream
  #37 = NameAndType        #49:#50        // println:(Ljava/lang/String;)V
  #38 = Utf8               LambdaMain2
  #39 = Utf8               java/lang/Object
  #40 = Methodref          #51.#52        // java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #41 = Methodref          #7.#53         // LambdaMain2.lambda$main$0:()V
  #42 = Utf8               hello
  #43 = Utf8               ()LHelloWorldLambda;
  #44 = Utf8               HelloWorldLambda
  #45 = Utf8               java/lang/System
  #46 = Utf8               out
  #47 = Utf8               Ljava/io/PrintStream;
  #48 = Utf8               java/io/PrintStream
  #49 = Utf8               println
  #50 = Utf8               (Ljava/lang/String;)V
  #51 = Class              #54            // java/lang/invoke/LambdaMetafactory
  #52 = NameAndType        #55:#59        // metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #53 = NameAndType        #22:#10        // lambda$main$0:()V
  #54 = Utf8               java/lang/invoke/LambdaMetafactory
  #55 = Utf8               metafactory
  #56 = Class              #61            // java/lang/invoke/MethodHandles$Lookup
  #57 = Utf8               Lookup
  #58 = Utf8               InnerClasses
  #59 = Utf8               (Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #60 = Class              #62            // java/lang/invoke/MethodHandles
  #61 = Utf8               java/lang/invoke/MethodHandles$Lookup
  #62 = Utf8               java/lang/invoke/MethodHandles
{
  public LambdaMain2();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   LLambdaMain2;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=2, args_size=1
         0: invokedynamic #2,  0              // InvokeDynamic #0:hello:()LHelloWorldLambda;
         5: astore_1
         6: aload_1
         7: invokeinterface #3,  1            // InterfaceMethod HelloWorldLambda.hello:()V
        12: return
      LineNumberTable:
        line 3: 0
        line 4: 6
        line 5: 12
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      13     0  args   [Ljava/lang/String;
            6       7     1 helloWorldLambda   LHelloWorldLambda;

  private static void lambda$main$0();
    descriptor: ()V
    flags: ACC_PRIVATE, ACC_STATIC, ACC_SYNTHETIC
    Code:
      stack=2, locals=0, args_size=0
         0: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #5                  // String hello menduo
         5: invokevirtual #6                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 3: 0
}
SourceFile: "LambdaMain2.java"
InnerClasses:
     public static final #57= #56 of #60; //Lookup=class java/lang/invoke/MethodHandles$Lookup of class java/lang/invoke/MethodHandles
BootstrapMethods:
  0: #27 invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
    Method arguments:
      #28 ()V
      #29 invokestatic LambdaMain2.lambda$main$0:()V
      #28 ()V
```

## 5.参考

https://www.infoq.cn/article/Invokedynamic-Javas-secret-weapon

https://blog.csdn.net/zxhoo/article/details/38387141

https://blog.csdn.net/kangkanglou/article/details/79422520

https://www.jianshu.com/p/3ae6efff1c96 静态分派和动态分派