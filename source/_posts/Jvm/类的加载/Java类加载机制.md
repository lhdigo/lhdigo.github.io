---
title: Java类加载机制
date: 2020-10-11 15:56:11
tags: jvm类加载
categories: [Jvm,类的加载]
---



# 什么是类的加载

- 类的加载指的是将**类的.class文件**中的**二进制数据**读入到**内存**中，将其放在运行时数据区的**方法区**内，然后在**堆**区创建一个 java.lang.Class对象，用来封装**类**在**方法区内的数据结构**。

  类的加载的最终产品是位于**堆区中的 Class对象**， Class对象封装了类在方法区内的数据结构，并且向Java程序员提供了访问方法区内的数据结构的接口。

<!--more-->

- 加载.class文件的方式
  - 从本地系统中直接加载
  - 通过网络下载.class文件
  - 从zip，jar等归档文件中加载.class文件
  - 从专有数据库中提取.class文件
  - 将Java源文件动态编译为.class文件



# 类的生命周期

1. 加载 Loading
2. 验证 Verification
3. 准备 Preparation
4. 解析 Resolution                  其中  2.验证  3.准备 4.解析为**连接阶段**
5. 初始化 Initialization
6. 使用 use
7. 卸载 Unloading

  

### 1. 加载 Loading 

> 查找并加载类的二进制数据

- 通过一个类的**全限定名**来获取其定义的二进制字节流

- 将这个字节流所代表的静态存储结构转化为**方法区的运行时数据结构**。

- 在Java**堆**中生成一个代表这个类的      java.lang.Class对象，作为对方法区中这些数据的访问入口



### 2. 验证 Verification

> 确保Class文件的字节流中包含的信息符合当前虚拟机的要求，但不是必须的。

1. 文件格式验证：验证字节流是否符合Class文件格式的规范；例如：是否以 0xCAFEBABE开头、主次版本号是否在当前虚拟机的处理范围之内、常量池中的常量是否有不被支持的类型。
2. 元数据验证：对字节码描述的信息进行语义分析（注意：对比javac编译阶段的语义分析），以保证其描述的信息符合Java语言规范的要求；例如：这个类是否有父类，除了java.lang.Object之外。
3. 字节码验证：通过数据流和控制流分析，确定程序语义是合法的、符合逻辑的。
4. 符号引用验证：确保解析动作能正确执行。

它对程序运行期没有影响，如果所引用的类经过反复验证，那么可以考虑采用 -Xverifynone参数来关闭大部分的类验证措施，以缩短虚拟机类加载的时间。

 

### 3. 准备 Preparation

> 为类的静态变量分配内存，并将其初始化为默认值

准备阶段是正式为类变量分配内存并设置类变量初始值的阶段，这些内存都将在**方法区**中分配。对于该阶段有以下几点需要注意：

1. 这时候进行内存分配的仅包括**类变量（static）**，而不包括实例变量，实例变量会在对象实例化时随着对象一块分配在**Java堆**中。
2. 这里所设置的初始值通常情况下是数据类型默认的零值（如0、0L、null、false等），而不是被在Java代码中被显式地赋予的值。

> 假设一个类变量的定义为： public static int value=3；
>
> 那么变量value在准备阶段过后的**初始值为0**，而不是3，因为这时候尚未开始执行任何Java方法，而把value赋值为3的 public static 指令是在程序编译后，存放于类构造器 <clinit>（）方法之中的，所以把value赋值为3的动作将在初始化阶段才会执行。
>

 

### 4. 解析 Resolution

> 把类中的符号引用转换为直接引用
> 符号引用就是一组符号来描述目标，可以是任何字面量。
> 直接引用就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄。

虚拟机将常量池内的符号引用替换为直接引用的过程，解析动作主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用点限定符7类符号引用进行



### 5. 初始化 Initialization

初始化，为类的静态变量赋予正确的初始值，JVM负责对类进行初始化，主要对类变量进行初始化。

Java中对类变量进行初始值设定有两种方式：

1. 声明类变量是指定初始值
2. 使用静态代码块为类变量指定初始值

 

JVM初始化步骤：

1. 假如这个类还没有被加载和连接，则程序先加载并连接该类
2. 假如该类的直接父类还没有被初始化，则先初始化其直接父类
3. 假如类中有初始化语句，则系统依次执行这些初始化语句

 

类初始化时机：只有当对类的主动使用的时候才会导致类的初始化，类的主动使用包括以下6种：

- 创建类的实例，也就是new的方式
- 访问某个类或接口的静态变量，或者对该静态变量赋值
- 调用类的静态方法
- 反射（如 Class.forName(“com.shengsiyuan.Test”)）
- 初始化某个类的子类，则其父类也会被初始化
- Java虚拟机启动时被标明为启动类的类（ JavaTest），直接使用 java.exe命令来运行某个主类



### 6. 结束生命周期 

Java虚拟机将结束生命周期：

1. 执行了 System.exit()方法
2. 程序正常执行结束
3. 程序在执行过程中遇到了异常或错误而异常终止
4. 由于操作系统出现错误而导致Java虚拟机进程终止



# 类加载器

```java
public class Classloadertest{
    public static void main(String[] args){
        ClassLoader Loader = Thread.currentThread().getContextClassLoader();
        System.out.println(Loader);
        System.out.println(Loader.getParent());
        System.out.println(Loader.getParent().getParent());
	}
}
```

运行后，输出结果:

```java
sun.misc.Launcher$AppClassLoader@18b4aac2
sun.misc.Launcher$ExtClassLoader@135fbaa4
null
```



从上面的结果可以看出，并没有获取到 `ExtClassLoader`的父Loader，原因是 `BootstrapLoader`（引导类加载器）是用C语言实现的，找不到一个确定的返回父Loader的方式，于是就返回`null`。

这几种类加载器的层次关系如下图所示：

![双亲委派机制流程图](https://i.loli.net/2020/10/19/ClOaoyvA1zNLPp4.png)



> 这里父类加载器并不是通过继承关系来实现的，而是采用组合实现的。
>
>

### 站在Java虚拟机的角度来讲，只存在两种不同的类加载器：

1. 启动类加载器：它使用C++实现（这里仅限于Hotspot，也就是JDK1.5之后默认的虚拟机，有很多其他的虚拟机是用Java语言实现的），是虚拟机自身的一部分；
2. 所有其它的类加载器：这些类加载器都由Java语言实现，独立于虚拟机之外，并且全部继承自抽象类 `java.lang.ClassLoader`，这些类加载器需要**由启动类加载器加载到内存中**之后才能去加载其他的类。

### 站在Java开发人员的角度来看，类加载器可以大致划分为以下三类：

**启动类加载器**： `BootstrapClassLoader`，负责加载存放在 `JDK\jre\lib`(JDK代表JDK的安装目录，下同)下，或被 `-Xbootclasspath`参数指定的路径中的，并且能被虚拟机识别的类库（如rt.jar，所有的java.开头的类均被 `BootstrapClassLoader`加载）。**启动类加载器是无法被Java程序直接引用的**。

这一步会加载关键的一个类：`sun.misc.Launcher`。这个类包含了两个静态内部类：

**扩展类加载器**： `ExtensionClassLoader`，该加载器由 `sun.misc.Launcher$ExtClassLoader`实现，它负责加载 `JDK\jre\lib\ext`目录中，或者由 `java.ext.dirs`系统变量指定的路径中的所有类库（如javax.开头的类），开发者可以直接使用扩展类加载器。
**应用程序类加载器**： `ApplicationClassLoader`，该类加载器由 `sun.misc.Launcher$AppClassLoader`来实现，它负责加载用户类路径（ClassPath）所指定的类，开发者可以直接使用该类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。



**不同类加载器加载的同一个类文件不相等。**

应用程序都是由这三种类加载器互相配合进行加载的，如果有必要，我们还可以加入自定义的类加载器。因为JVM自带的ClassLoader只是懂得从本地文件系统加载标准的java class文件，因此如果编写了自己的ClassLoader，便可以做到如下几点：

- 1、在执行非置信代码之前，自动验证数字签名。

- 2、动态地创建符合用户特定需要的定制化构建类。

- 3、从特定的场所取得java class，例如数据库中和网络中。


### 双亲委派机制

![双亲委派机制流程](https://i.loli.net/2020/10/19/Nbu4arm1oE7jclz.png)

- 当前ClassLoader首先从**自己**已经加载的类中查询是否此类已经加载，如果已经加载则直接返回原来已经加载的类。每个类加载器都有自己的加载缓存，当一个类被加载了以后就会放入缓存，等下次加载的时候就可以直接返回了。
- 一个类加载器收到了类加载请求，不会自己立刻尝试加载类，而是把请求委托给父加载器去完成，每一层都是如此，所有的加载请求最终都传递到最顶层的类加载器进行处理；
- 如果父加载器不存在，那么尝试判断有没有被**启动类加载器**加载；
- 如果**启动类加载器**也没有加载此类，则自己尝试加载它，并放入自己加载类的缓存中。

#### 如何判为什么断类是否重复加载

JVM比较

1. **类是否相等**
2. 加载这两个类的**类加载器**是否相等，

只有同时满足条件，两个类才能被认定是相等。



#### 为什么要有这么复杂的双亲委派机制

> 为什么要有这么复杂的双亲委派机制？
>
>   如果没有这种机制，我们就可以篡改启动类加载器中需要的类了，如，修自己编写一个`java.lang.Object`用自己的类加载器进行加载，系统中就会存在多个Object类，这样Java类型体系最基本的行为也就无法保证了。



#### 双亲委派模型特点：

- 父加载器中加载的类对于所有子加载器可见。
- 子类之间各自加载的类对于各自是不可间的（达到隔离效果）。

- 类是唯一的，没有重复的类。



# 类的加载

### 类加载有三种方式：

- 1、由 new 关键字创建一个类的实例（静态加载）

- 2、通过Class.forName()方法动态加载，通过反射加载类型，并创建对象实例

  如：Class clazz ＝ Class.forName（“Dog”）；
  Object dog ＝clazz.newInstance（）；

- 3、通过ClassLoader.loadClass()方法动态加载

  如：Class clazz ＝ classLoader.loadClass（“Dog”）；
  Object dog ＝clazz.newInstance（）；



  ```java
  public class loaderTest  {
      public static void main(String[] args) {
          try {
              //class.forName 加载方式
              Class<?> classForName = Class.forName("com.lhdigo.TestClass");
              Constructor<?> constructor1 = classForName.getConstructor(String.class);
              constructor1.newInstance("加载方式Class.forName:有一个参数的构造器");
  
              //loader加载方式
              ClassLoader classLoader = ClassLoader.getSystemClassLoader();
              Class<?> classLoaderClass = classLoader.loadClass("com.lhdigo.TestClass");
              Constructor<?> constructor2 = classLoaderClass.getConstructor(String.class);
              constructor2.newInstance("加载方式ClassLoader:有一个参数的构造器");
          }catch (Exception e){
              
          }
      }
  }
  ```


### 三者的区别：

- 1和2使用的类加载器是相同的，都是当前类加载器。（即：`this.getClass().getClassLoader()`）。
- 3由用户**指定类加载器**。如果需要在当前类路径以外寻找类，则只能采用第3种方式。第3种方式加载的类与当前类分属不同的命名空间。



### Class.forName与ClassLoader.loadClass区别

首先，我们知道，Class的装载包括3个步骤：

**加载（loading）,连接（link）,初始化（initialize）**.(连接包括：验证 准备 解析)

- `Class.forName(className)`实际上是调用

  `Class.forName(className, true, this.getClass().getClassLoader())`。

第二个参数，是指Class被loading后是不是必须被初始化。

- `ClassLoader.loadClass(className)`实际上调用的是`ClassLoader.loadClass(name, false)`，第二个参数指Class是否被link。

**总结**：

- `Class.forName`得到的`class`是已经**初始化**。
- `ClassLoader.loadeClass`得到的`class`是还**没有连接**的。



# 自定义类加载器

### **为什么要自定义类加载器？**

1. 加密：.class文件可以轻易的被**反编译**，如果你需要把自己的代码进行加密以防止反编译，可以先将编译后的代码用某种加密算法加密，类加密后就不能再用Java的ClassLoader去加载类了，这时就需要自定义ClassLoader在加载类的时候先解密类，然后再加载。
2. 从非标准的来源加载代码：如果你的.class文件是放在数据库、甚至是在云端，就可以自定义类加载器，从指定的来源加载类。

如你的应用需要通过网络来传输 Java 类的字节码，为了安全性，这些字节码经过了加密处理。这个时候你就需要自定义类加载器来从某个网络地址上读取加密后的字节代码，接着进行解密和验证，最后定义出在Java虚拟机中运行的类



### 自定义类加载器步骤：

1 继承ClassLoader抽象类

2 重写findClass() 

3 重写的findClass()中调用defineClass()

```java
package com.lhdigo;

import java.io.*;

public class MyClassLoader extends ClassLoader {
    private String root;

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classData = loadClassData(name);
        if (classData == null) {
            throw new ClassNotFoundException();
        } else {
            return defineClass(name, classData, 0, classData.length);
        }
    }

    private byte[] loadClassData(String className) {
        String fileName = root + File.separatorChar + className.replace('.',File.separatorChar )+ ".class";
        try{
            InputStream ins = new FileInputStream(fileName);
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            int bufferSize = 1024;
            byte[] buffer = new byte[bufferSize];
            int length = 0 ;
            while ((length = ins.read(buffer))!= -1){
                baos.write(buffer,0,length);
            }
            return baos.toByteArray();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    public String getRoot() {
        return root;
    }

    public void setRoot(String root) {
        this.root = root;
    }

    public static void main(String[] args) {
        MyClassLoader myClassLoader  =  new MyClassLoader();
        myClassLoader.setRoot("E:\\temp");
        Class<?> testClass  = null;
        try {
            testClass = myClassLoader.loadClass("com.lhdigo.TestClass");
            Object o = testClass.newInstance();
            System.out.println(o.getClass().getClassLoader());
        } catch (ClassNotFoundException | IllegalAccessException | InstantiationException e) {
            e.printStackTrace();
        }
    }
}

```

自定义类加载器的核心在于对字节码文件的获取，如果是加密的字节码则需要在该类中对文件进行解密。

- 这里传递的文件名需要是类的全限定性名称，即 `com.lhdigo.TestClass`格式的，因为 defineClass 方法是按这种格式进行处理的。
- 2、最好不要重写`loadClass`方法，因为这样容易破坏双亲委托模式。
- 3、这类`TestClass`类本身可以被 `AppClassLoader`类加载，因此我们不能把 `com/paddx/test/classloading/Test.class`放在类路径下。否则，由于双亲委托机制的存在，会直接导致该类由 `AppClassLoader`加载，而不会通过我们自定义类加载器来加载。