---
title: JVM学习笔记1_Java内存区域与内存溢出异常
date: 2019-03-30 22:07:16
tags:
    - Java
    - JVM
    - 技术
---

# JVM学习笔记1_Java内存区域与内存溢出异常
## 参考资料
周志明《深入理解Java虚拟机》

## Java内存区域与内存溢出异常
<!--more-->

Java运行时数据区域划分:  
>程序计数器（PC）
>>当前线程所执行的字节码的地址指示器，概念模型当中字节码解释器工作时就是通过改变这个计数器的值来确定下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器完成

>Java虚拟机栈
>>虚拟机栈描述的是Java方法执行的内存模型，每个方法在执行的时候会建立一个栈帧，存储局部变量表、操作数栈、动态链接、方法出口等信息。每一个方法从调用直至完成的过程对应着一个栈帧在虚拟机栈中从入栈到出栈的过程  
此外虚拟机栈当中还有局部变量表，存放了编译期可知的各种基本数据类型、对象引用（引用指针或者句柄，并非是对象本身）和returnAddress（指向字节码指令地址）  
long和double占据两个Slot，其它基本类型占用一个

>本地方法栈
>>作用与虚拟机栈类似，执行Native方法服务

>Java堆
>>存放对象实例及数组的主要区域，也是GC主管的区域  
可以细分为新生代和老年代  
可以通过-Xms（启动时分配内存）和-Xmx（运行时最大分配内存）进行大小的调节  
此外-Xss用于设置线程启动时分配的空间大小
（HotSpot当中常量）

>方法区
>>存放已被虚拟机加载的类信息、常量、静态变量等与类有关的信息  
PS：Class对象也是对象，所以是在堆里而不是方法区~

>运行时常量池
>>存放编译期生成的各种字面量和符号引用，相对于Class常量池具备动态性（String的intern()方法）
(1.7以后这一部分也被放到了堆当中)

>直接内存
>>通过native函数库直接分配堆外内存（用于基于通道和缓冲区的I/O方式），通过Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作，避免在Java堆和Native堆当中来回复制数据  
同时也是容易被忽视的导致OOM异常的区域

对象的创建（虚拟机中的普通对象创建，不包括数组和Class对象等）：
>类加载检查：首先检查参数能否在常量池当中定位到一个类的符号引用，并且检查这个符号引用的类是否已经被加载验证准备解析初始化（后续会详述）  
分配内存：连续内存的话是指针碰撞，不连续内存的话是空闲列表（取决于使用的GC的特性）  
线程安全的考虑：使用CAS/TLAB（每个线程有一个，哪个需要给哪个）  
内存空间初始化（对象头除外）  
对象设置：比如是哪个类的实例、元数据信息的定位、哈希码、GC年龄等  
执行init方法

对象的内存布局：
>对象头，包括两部分信息：  
>>1.存储运行时数据，包括：
>>>哈希码  
GC分代年龄  
锁状态标志  
线程持有的锁  
偏向线程ID  
偏向时间戳  
etc  

>>2.类型指针，确定对象是哪个类的实例。另外Java数组还需要记录数组长度

>实例数据部分：
>>各种类型的字段内容  
存储顺序受到虚拟机分配策略参数（FieldsAllocationStyle）影响  
HotSpot当中默认的分配顺序是longs/doubles、ints、shorts/chars、bytes/booleans、
oops（Ordinary Object Pointers），相同宽度的字段总是被分配到一起

>对齐填充：如同名字所述，只是管理系统要求起始地址是8字节整数倍的时候调整用的占位符，根据情况可有可无

对象的访问定位：
>句柄：指向句柄池，reference当中存储稳定的句柄地址，对象被移动（比如说垃圾收集是移动对象）时不需要改变reference的指向。  
>直接指针：直接访问，快，节省开销

OOM异常的各种特殊技巧：  
1.Java堆溢出
```java
/**
 * VM Aegs: -Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
 */
public class HeapOOM{
    static class OOMObject{
    }

    public static void main(String[] args){
        List<OOMObject> list = new ArrayList<OOMObeject>();
        while(true){
            list.add(new OOMObject());
        }
    }
} 
```
2.虚拟机栈和本地方法栈溢出
```java
/**
 *VM args: -Xss128k
 */
public class JavaVMStackSOF{
    private int stackLength = 1;
    public void stackLeak(){
        stackLength++;
        stackLeak();
    }
    public static void main(String[] args) throws Throwable{
        JavaVMStackSOF oom = new JavaVMStackSOF();
        try{
            oom.stackLeak();
        }catch(Throwable e){
            System.out.println("stack length:" + oom.stackLength);
            throw e;
        }
    }
}
```

```java
/**
 *VM Args: -Xss2M
 */
public class JavaVMStackOOM{
    private void dontStop(){
        while(true){

        }
    }
    public void stackLeakByThread(){
        while(true){
            Thread thread = new Thread(new Runnable(){
                @Override
                public void run(){
                    dontStop();
                }
            });
            thread.start();
        }
    }
    public static void main(String[] args)throws Throwable{
        JavaVMStackOOM oom = new JavaVMStackOOM();
        oom.stackLeakByThread();
    }
}
//友情提示，不要在Windows上跑这段，凉过一次了Orz
```
方法区和运行时常量池溢出:
```java
/**
 * VM Args: -XX:PremSize=10M -XX:MaxPermSize=10M
 */
//PS：取消了PermGen的JDK1.7以上不会出现OOM异常
//原因是intern()不再复制实例，只记录首次出现的实例
public class RuntimeConstantPoolOOM{
    public static void main(String[] args){
        List<String> list = new ArrayList<String>();
        int i = 0;
        while(true){
            list.add(String.valueOf(i++).intern());
        }
    }
}
```

```java
/**
 * VM Args: -XX:PermSize=10M -XX:MaxPermSize=10M
 */
public class JavaMethodAreaOOM{
    public static void main(Stirng[] args){
        while(true){
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperClass(OOMObject.class);
            enhancer.setUseCache(false);
            enhancer.setCallback(new MethodInterceptor(){
                public Object intercpet(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable{
                    return proxy.invokeSuper(obj, args);
                }
            });
            enhancer.create();
        }
    }
    static class OOMObject{

    }
}
```
本机直接内存溢出：
```java
/**
 * VM Args: -Xmx20M -XX:MaxDirectMemorySize=10M
 */
public class DirectMemoryOOM{
    private static final int _1MB = 1024*1024;

    public static void main(Stinrg[] args) throws Exception{
        Field unsafeField = Unsafe.class.getDeclaredFields()[0];
        unsafeField.setAccessible(true);
        Unsafe unsafe = (Unsafe) unsafeField.get(null);
        while(true){
            unsafe.allocateMemory(_1MB);
        }
    }
}
```
