---
title: JVM学习笔记_5_class文件_属性表补充
date: 2019-08-05 08:48:55
tags:
    - Java
    - JVM
    - 技术
---
# JVM 属性表--java class 补充
## 参考资料
《深入理解Java虚拟机》
<!--more-->

## 没有严谨的顺序要求
* 可以写入自己定义的属性信息，但是jvm会忽略它不认识的信息

## 主要属性
* 主要的属性如下：

属性名|使用位置|含义
--|--|--
Code|方法表|Java代码编译成的字节码指令
ConstantValue|字段表|final关键字定义的常量值
Deprecated|类、方法、字段表|被声明为Deprecated的方法和字段
Exceptions|方法表|方法抛出的异常
EnclosingMethod|类文件|仅当一个类为局部类或者匿名类时才能拥有这个属性，这个属性用于标识这个类所在的外围方法
InnerClasses|类文件|内部类列表
LineNumberTable|Code属性|Java源码的行号与字节码指令的对应关系
LocalVariableTable|Code属性|JDK 1.6中新增的属性，供新的类型检查验证其（TypeChecker)检查处理目标方法的局部变量和操作数栈所需类型是否匹配
StackMapTable|Code属性|JDK1.6新增属性，供新的类型检查验证器（TypeChecker）检查和处理目标方法的局部变量和操作数栈所需要的类型是否匹配
Signuture|类、方法表、字段表|JDK1.5中新增的属性，这个属性用于支持泛型情况下的方法签名（防止泛型类型信息在被擦除后签名混乱）
SourceFile|类文件|记录源文件名称
SourceDebugExtension|类文件|JDK1.6中新增的属性，SourceDebugExtension属性由于存储额外的调试信息。（为非Java语言编写却要编译运行在JVM的程序提供进行调试的信息）
Synthetic|类、方法表、字段表|表示方法或字段为编译器自动生成的
LocalVariableTypeTable|类|JDK1.5，用特征签名代替描述符
RuntimeVisibleAnnotations|类、方法、字段表|JDK1.5，支持动态注解，注明注解在运行（反射调用）时的可见性
RuntimeInVisibleAnnotation|类、方法、字段表|JDK1.5，与Visible相反，注明注解的不可见性
AnnotationDefault|方法表|JDK1.5，记录注解类元素的默认值
BootstrapMethods|类文件|JDK1.7，保存invokeddynamic指令引用的指导方法限定符

## 结构
类型|名称|数量
--|--|--
u2|attribute_name_index|1
u4|attribute_length|1
u1|info|attribute_length

## 各种属性的大致介绍
其实书里还给了一些具体的例子啥的…但是我不太想抄了QAQ，所以就把一些自己觉得比较重要的部分摘出来，如果大家感兴趣的话可以去搜索相关的博客或者直接阅读原著呀（参考链接当中有的）
### Code属性
Java程序方法体中的代码经过Javac编译器处理后，最终变为字节码指令存储在Code属性当中，Code属性出现在方法表的集合属性当中，但并非所有方法表都必须有该属性（接口/抽象方法可以没有）
#### Code属性表结构
类型|名称|数量
--|--|--
u2|attribute_name_index|1
u4|attribute_length|1
u2|max_stack|1
u2|max_locals|1
u4|code_length|1
u1|code|code_length
u2|exception_table_length|1
exception_info|exception_table|exception_table_length
u2|attributes_count|1
attribute_info|attributes|attibutes_count

#### Code_有关说明
* code用于存储字节码
* code_length是u4，理论上可以达到2^32-1，但是实际上虚拟机限制了它的长度到65535

#### Code_Java虚拟机执行字节码的体系结构
* 基于栈
* 某些指令带有参数
* `this`关键字会变成一个局部变量传入

#### Code_异常表
* Java异常以及Finally处理机制是通过异常表
* 而不是跳转机制
（TBC…每一个都要抄上去感觉精神上有点痛苦Orz）

### Exception属性
* PS：这个属性和异常表没有关系，不要产生奇怪的误会啊啊啊啊啊
* 那么这个有什么用呢？--（自问自答）列举出方法中可能抛出的受查异常

### LineNumberTable属性
* 描述源代码行号与字节码行号（字节码偏移量）之间的关系
* 默认生成，但不必要，可以在使用`javac`指令时分别使用`-g:none`取消生成，或者`-g:line`要求生成（不是已经默认了吗Orz）
* 如果不生成，抛异常的时候不会显示代码行号，也没有办法设置断点调试程序

### LocalVariableTable
总体来说和LineNumberTable的作用方式有点像：

* 描述栈帧当中局部变量表的变量与Java源码中定义的变量之间的关系
* 默认生成，但不必要，可以在使用`javac`指令时分别使用`-g:none`取消生成，或者`-g:vars`要求生成（不是已经默认了吗Orz）
* 不生成的时候，别人引用的时候不会显示参数名（用arg0/arg1之类的代替），调试期也无法根据参数名获取参数值
* 主要属性有start_pc+index（确定局部变量的作用范围）etc...

### LocalVariableTypeTable
* 与LocalVariableTable很像
* 把LocalVariableTable的descriptor_index替换成了字段的特征签名（signature），因为引入泛型之后描述符难以准确地描述泛型类型了

### SourceFile
记录class文件的源码文件名称

### ConstantValue
* 通知虚拟机自动为静态变量赋值
* 类变量（static）才可以使用这项属性
* 非类变量在实例构造器`<init>`方法中实现初始化，类变量则使用`<cinit>`或者是设置ConstantValue
* 通常是给final static（类常量使用的），但没有硬限制，具体策略取决于JVM自己的实现
* 一般只用于基本类型和String（自身能力有限）

### Innerclass
* 记录内部类与宿主类之间的关联
* 匿名内部类的`inner_name_index`值为0

### Deprecated
* 表明作者已经不再推荐使用这种操作了
* 由`@Deprecated`注解生成

### Synthetic
表示该字段/方法不是由源码生成的而是JVM添加的（例如Bridge Method）

### StackMapTable
* 字节码验证阶段用于检查验证器
* 用验证类型的检查替代了运行期通过数据流分析来确认字节码行为逻辑合法性的步骤
* 太过于硬核了，感兴趣的同学自己去看看把，接下来可能也会研究一下

### Signature
* 记录泛型签名信息
* 使得使用擦除法的Java反射API也能获取泛型信息

### BootstrapMethods
* 保存invokeDynamic指令引用的引导方法限定符