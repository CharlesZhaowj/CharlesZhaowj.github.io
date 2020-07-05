---
title: JVM学习笔记_4_class文件
date: 2019-07-22 09:04:35
tags:
    - Java
    - JVM
    - 技术
---
# JVM学习笔记4_class文件结构

## 参考资料
《深入理解Java虚拟机》
鸭子船长_命令查看java的class字节码文件 - https://www.cnblogs.com/zl1991/p/6904393.html

<!--more-->

## 无关性
### 平台无关性
所有平台统一使用相同的存储格式--字节码
### 语言无关性
其他语言（如Groovy、Jython、Scala等）也可以在JVM上运行

## Class文件
### 查看class文件的命令
* `javac –verbose HelloWorld.java`：可以看到虚拟机编译时完成的操作
* `java –verbose HelloWorld`：可以看到虚拟机运行一个程序时加载的jar包
* `javap –c HelloWorld`：可以查看字节码，从中可以得到各种变量的信息等等
* `javap –verbose`：可以看到更加清楚的信息（文件路径、版本、常量池和方法等）

### Class文件的结构
1. Class文件是以组以8字节位基础单位的二进制流，没有分隔符
2. 遇到需要占用8位字节以上空间的数据项时，则会按照高位在前（大端）分割成多个8位字节存储
3. 采用伪结构存储数据，只有两种数据类型：无符号数和表

#### Class文件格式
类型|名称|数量
--|--|--
u4|magic|1
u2|minor_version|1
u2|major_version|1
u2|constant_pool_count|1
cp_info|constant_pool|constant_pool_count-1
u2|access_flags|1
u2|this_class|1
u2|super_class|1
u2|interfaces_count|1
u2|interfaces|interfaces_count
u2|fields_count|1
field_info|fields|fiels_count
u2|methods_count|1
method_info|methods|methods_count
u2|attributes_count|1
attribute_info|attributes|attributes_count

说明：没有分隔符，所以顺序、数量、字节序都是严格限定的

#### 无符号数
1. 基本的数据类型，以u1、u2、u4、u8分别代表相应字节的无符号数
2. 用于描述数字、索引引用、数量值或按照UTF-8编码构成字符串值

#### 表
1. 由多个无符号数或者其他表作为数据项构成的复合数据类型
2. 以"_info"结尾

### 类型的顺序和数量都已经事先定好了
1. 头四个字节：魔数`0xCAFEBABE`（在Java还叫作Oak的时候就被作者硬点了。）
2. 接着四个字节：Class文件的版本号。--第5、6字节是主版本号（Major Version），第7、8字节是次版本号（Minor Version）。主版本号就是生成class文件的JDK版本号（也是这个class文件最高能运行的版本号），次版本号就是其最低支持的JDK版本，低于次版本号对应的JDK的虚拟机都会拒绝运行这个class文件。
    * 至于具体那个数对应哪个版本号，请大家自行百度摸♂索
3. 之后是常量池入口、常量池可以理解位Class文件的资源仓库，是class文件结构当中与其他项目关联最多的数据类型。同时它也是Class文件当中第一个出现的表类型（多个无符号数或其他表组合而成的符合数据类型）：
    1. u2类型数据：常量池容量计数值（从1开始），表示常量池当中的常量个数（比如说计数值是22的时候，就有21个常量）
    2. 存放字面量+符号引用
        * 字面量：
            * 文本字符串
            * final常量值等
        * 符号引用：
            * 类和接口的全限定名
            * 字段的名称和描述符
            * 方法的名称和描述符
        * （虚拟机运行时，需要从常量池获得对应的符号引用，再在类创建时或运行时解析、翻译到具体的内存地址当中）
        * 常量池当中每一项都是一个表：首先有一个u1标志位，然后又有各自的结构
    3. 之后是访问标志（access_flags）：识别类/接口层次的访问信息，比如类/接口，是否为public类型，是否为abstract类型，类是否被声明为final等。
    4. 然后是类索引、父类索引和接口索引：
        ```
        类索引：用于确定本类的全限定名（包名+外部类名$内部类名）
        父类索引：确定类父类的全限定名（只有一个，Object没有）
        接口索引：确定实现的接口（如果本身是接口就是实现了哪些接口）
        ```
    5. 然后是字段表集合
        ```
        包括了字段的各种信息（包括作用域（public/private/protected）、static修饰符、final可变性、volatile并发可见性、transient可否序列化、字段的数据类型、字段名称），用标志位表示修饰符字段的信息（就是只有0/1两种情况的像是static这种的）
        字段名字和数据类型则用常量池中的常量来表述
        数组的话每一位用一个前置中括号，例如[[Ljava/lang/String
        PS: 对于字节码来说，如果两个字段的描述符不一致，那字段重名就合法（但是Java语法是不允许的）
        ```
    6. 然后是方法表集合，这个的构成和字段表有很多相似之处：
        ```
        访问标志
        名称索引
        描述符索引
        属性表集合
        ```
        * Q：方法里的代码去哪了。 A：存放在方法属性表
        * 重载方法的时候，需要拥有与原方法不同的特征签名（一个方法中各个参数在常量池中字段符号引用的集合），Java当中返回值不包含在特征签名里面，但是class文件当中是包含的（emmm…
    7. 然后是属性表集合。class文件、字段表、方法表都可以携带自己的属性表集合，用于描述某些特定的信息。这个的讲解太长了，所以单列一个。