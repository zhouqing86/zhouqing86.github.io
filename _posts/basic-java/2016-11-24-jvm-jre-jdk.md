---
layout: post
comments: false
categories: Java基础
date:   2016-11-24 18:30:54
title: JVM, JRE, JDK解释
---

<div id="toc"></div>

## JVM, JRE, JDK的关系

下面两幅图片简单说明了 **JVM** (Java Virtual Machine, Java虚拟机), **JRE** (Java Runtime Environment, Java运行时环境), **JDK** (Java Development Kit, Java软件开发工具包)的关系。

{% include image.html url="/static/img/jvm-jre-jdk-1.jpg" description="图1" height="400px" inline="true" %}

{% include image.html url="/static/img/jvm-jre-jdk-2.jpg" description="图2" height="400px" inline="true" %}

图一中简单的说明了三者之间的从属关系:
- JRE = JVM + 核心基础类库
- JDK = JRE + 开发工具包

图二需要做一点解释：
- JDK里的Java编译器将Java源码文件编译成Java字节码
- JVM负责读取Java字节码并使得程序运行在硬件平台上
- JRE是JVM的实践，是Java程序的运行环境

## JVM
- JVM是一个虚拟计算机，它包括一套字节码指令集、一组寄存器、一个栈、一个垃圾回收堆和一个存储方法域。
- JVM对Java编程语言是一无所知的，但它能识别.class格式的文件（包括字节码，符号表以及辅助信息，这意味着任何语言如果能编译成.class格式文件都能运行在JVM上。
- JVM屏蔽了与具体操作系统平台相关的信息，这也意味着JVM得适配各种操作系统，各种底层硬件设备, 目前不同的操作系统需要使用不同的JVM（当然我们在开发或运行Java程序时并不关注这些，只需要安装相应系统的JDK和JRE就可以了）。
- 目前市面上有很多不同厂商的JVM，大部分是免费的，目前主要流行的有HotSpot VM，J9 VM， Zing VM等，目前Oracle / Sun JDK、OpenJDK的各种变种用的都是HotSpot VM。
- JVM处于不断迭代发展之中，JDK8的HotSpot VM已经是以前的HotSpot VM与JRockit VM的合并版，也就是传说中的“HotRockit”，只是产品里名字还是叫HotSpot VM。

<br/>

### 内存管理
JVM在执行Java程序的过程中会把它所管理的内存划分为若干不同的数据区域，这些区域都有各自的用途以及创建和销毁的时间。这些区域包括：
- 栈，其为线程私有的内存区域，生命周期与线程相同。
- 堆，其为线程共享的内存区域，唯一目的是存放对象实例，其是垃圾收集器管理的主要区域。
- 方法区，其为线程共享的内存区域，存储JVM加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。
- 本地方法栈，其为线程私有的内存区域，为JVM使用到的Native方法服务.
- 程序计数器, 其为线程私有的内存区域，当前线程所执行的字节码的行号指示器，它是唯一一个没有规定任何OutOfMemoryError情况的区域.

<br/>
### GC(垃圾收集)
GC的目的在于清除不再使用的对象，GC需要通过相关算法来判断某对象是否可以被清除。GC的主要区域是堆，堆被划分为新生代和旧生代，JVM对新生代和旧生代采用不同的垃圾回收机制。
- 新生代GC
  - 串行GC(SerialGC)
  - 并行回收GC(ParallelScavenge)
  - 并行GC(ParNew)
- 旧生代GC
  - 串行GC(SerialMSC)
  - 并行GC(parallelMSC)
  - 并发GC(CMS)

<br/>
### JIT（即时编译器)
之前的Java程序是通过解释器解释执行的，字节码逐条解释，执行速度相对较慢。引入JIT的目的是为了提高程序的执行速度，JIT在程序运行时会把翻译过的机器码保存起来，已备下次使用，因此从理论上来说，采用该JIT技术可以接近以前纯编译技术。

JIT默认是启用的，JVM读入.class文件后，会将其发给JIT，JIT将字节码编译成本机机器代码。但是目前JIT并不是对所有的字节码进行编译，而是只把一些经常执行的字节码(Hot Spot Code, 热点代码)编译成本地平台相关的机器码。

HotSpot VM中内置了两个JIT编译器：Client Complier 和 Server Complier，分别用在客户端和服务端，目前主流的 HotSpot 虚拟机中默认是采用解释器与其中一个编译器直接配合的方式工作。

## JRE
- JRE包括Java虚拟机（jvm）、Java核心类库（主要是rt.jar)和支持文件
- JRE需要辅助软件"Java Plug-in"来在浏览器中运行applet

### rt.jar
JVM知道rt.jar的存在，不会对rt.jar包做检查。rt.jar中包含了所有的系统包，如java.lang, java.io。包括系统常用类如
- java.lang.String
- java.io.File
- java.lang.Thread
- java.util.ArrayList
- java.io.InputStream

### jvm.cfg
控制JVM的选择，此文件中的第一行配置为默认的JVM。
还能控制是选用Client Compiler还是Server Compiler的的JIT。

### Compact Profile
一个完整的JRE需要>100MB，不能部署和运行在小型设备上，但现在Java 8有个很好的特性，即[JEP161](http://openjdk.java.net/jeps/161),该特性定义了Java SE平台规范的一些子集，使java应用程序不需要整个JRE平台即可部署和运行在小型设备上。
- compact1, 10MB
- compact2, 17MB
- compact3, 24MB


## JDK
- JDK包含完整的JRE
- JDK包含了一批用于Java开发的组件，如
  - javac, 编译器
  - jdb, 调试工具
  - javadoc, 文档工具
- JDK中还包括各种样例文件来展示Java API中的各个部分
- JDK一般有三种版本: J2SE, J2EE, J2ME

### tools.jar
tools.jar包含支持JDK的工具和实用程序的非核心类：tools.jar来支持javac, javadoc, javah, javap等命令的运行。

### dt.jar
> Also includes dt.jar, the DesignTime archive of BeanInfo files that tell interactive development environments (IDE's) how to display the Java components and how to let the developer customize them for an application.

<br/>
## 参考资料
- [Java Programming Environment and the Java Runtime Environment (JRE)](https://docs.oracle.com/cd/E19455-01/806-3461/6jck06gqd/index.html)
- [What is difference between JDK, JRE and JVM?](https://www.quora.com/What-is-difference-between-JDK-JRE-and-JVM)
- [The Java® Virtual Machine Specification](https://docs.oracle.com/javase/specs/jvms/se8/html/)
- [Comparison of Java virtual machines](https://en.wikipedia.org/wiki/Comparison_of_Java_virtual_machines)
- [精简的JRE详解](https://my.oschina.net/benhaile/blog/211804)
- [JDK and JRE File Structure](http://docs.oracle.com/javase/7/docs/technotes/tools/solaris/jdkfiles.html)

<script type="text/javascript">
$(document).ready(function() {
    $('#toc').toc({ listType: 'ul', title: "<i>目录</i>" });
});
</script>
