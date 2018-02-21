---
title: Java类加载过程
date: 2018-02-20 19:28:54
categories:
- Java
tags: 
- java
---
Java运行的是这样的，首先Java编译器将我们的源代码编译成为字节码，然后由JVM将字节码load到内存中，接着我们的程序就可以创建对象了

## 类加载过程
一个java文件被加载到内存中需要经历三个阶段 `加载-连接（验证+准备+解析）->初始化`
![java-classload](http://p0kzweqn4.bkt.clouddn.com/java-classload.png)
<!-- more -->
* 加载  
查找并加载类的二进制数据
* 连接  
验证：确保被加载的类的正确性  
准备：为类的静态变量分配内存，并将其初始化为默认值  
解析：把类中的符号引用转换为直接引用
* 初始化  
为类的静态变量赋予正确的初始值（自主赋予的值）

## 类的初始化
Java程序对类的使用方式可分为2种，主动使用和被动使用。所有的Java虚拟机实现必须在每个类或接口被Java程序**首次主动使用时才初始化他们**

### 类的初始化触发（主动调用）
1. 创建类的实例，也就是new一个对象 
2. 访问某个类或接口的静态变量，或者对该静态变量赋值
3. 调用类的静态方法
4. 反射（Class.forName("cc.zxian")）
5. 初始化一个类的子类（会首先初始化子类的父类）
6. Java虚拟机启动时被标明为启动类的类（文件名和类名相同，含有main方法并且是启动方法的类）

### 类的初始化顺序
1. 如果这个类还没有被加载和链接，那先进行加载和链接 
2. 假如这个类存在直接父类，并且这个类还没有被初始化（注意：在一个类加载器中，类只能初始化一次），那就初始化直接的父类（不适用于接口） 
3. 加入类中存在初始化语句（如static变量和static块），那就依次执行这些初始化语句。 
4. 总的来说，初始化顺序依次是：（静态变量、静态初始化块）–>（变量、初始化块）–> 构造器；如果有父类，则顺序是：父类static方法 –> 子类static方法 –> 父类构造方法- -> 子类构造方法

## 类加载器
### 加载器介绍
1. BootstrapClassLoader（启动类加载器）  
负责加载$JAVA_HOME中jre/lib/rt.jar里所有的class，加载  
System.getProperty("sun.boot.class.path")所指定的路径或jar。 
2. ExtensionClassLoader（标准扩展类加载器）  
负责加载java平台中扩展功能的一些jar包，包括$JAVA_HOME中jre/lib/*.jar或-Djava.ext.dirs指定目录下的jar包。加载System.getProperty("java.ext.dirs")所指定的路径或jar。 
3. AppClassLoader（系统类加载器）  
负责加载classpath中指定的jar包及目录中class 
4. CustomClassLoader（自定义加载器）  
属于应用程序根据自身需要自定义的ClassLoader，如tomcat、jboss都会根据j2ee规范自行实现。

![类加载器](http://p0kzweqn4.bkt.clouddn.com/classloader.png)

### 类加载器的顺序 
1. 加载过程中会先检查类是否被已加载，检查顺序是自底向上，从Custom ClassLoader到BootStrap ClassLoader逐层检查，只要某个classloader已加载就视为已加载此类，保证此类只所有ClassLoader加载一次。而加载的顺序是自顶向下，也就是由上层来逐层尝试加载此类。 
2. 在加载类时，每个类加载器会将加载任务上交给其父，如果其父找不到，再由自己去加载。 
3. Bootstrap Loader（启动类加载器）是最顶级的类加载器了，其父加载器为null。

## 参考
1. [java中类的加载顺序介绍(ClassLoader)](http://blog.csdn.net/eff666/article/details/52203406)
2. [深入剖析Classloader(一)--类的主动使用与被动使用](http://yhjhappy234.blog.163.com/blog/static/3163283220115573911607/)

---