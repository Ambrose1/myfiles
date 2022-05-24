[toc]

# 消息转发机制

问： 什么是消息转发机制？

问： 消息转发失败了怎么办？



1. 消息动态解析
2. 消息接受者重定向
3. 消息重定向



# id /isa/ instance 区别

+ id 是万能指针，源码中id 是objc_object 结构体的指针，可以指向任意实例对象。

+ Objc_object 是根对象，结构体中只有一个isa指针。

  ```cpp
  // Represents an instance of a class
  struct objc_object {
    Class _Nonnull isa OBJC_ISA_AVAILABILITY;
  };
  ```

+ 类对象objc_class继承于obj_object



# 为什么要设计元类 meta class

+ 类对象、元类对象能够复用消息发送流程机制。
+ 单一职责原则



## class_cpyIvarList 和class_copyPropertyList 的区别



# isa

https://www.jianshu.com/p/6438471835da

+ isa 指针保存着指向类对象的内存地址class。

+ 类对象全局只有一个，每个类创建出来的对象都会默认有一个isa属性.

+ 通过class可以查询到这个对象的属性和方法、协议等。

+ isa指针占据8字节内存。

  ```c
  union isa_t {
      isa_t() { }
      isa_t(uintptr_t value) : bits(value) { }
  
      Class cls;
      uintptr_t bits;
  #if defined(ISA_BITFIELD)
      struct {
          ISA_BITFIELD;  // defined in isa.h
      };
  #endif
  };
  
  ```

## 什么是Tagged Pointer

tagpointer 类型对象没有指针。

# assert 

`assert` 宏定义在`assert.h` 中, 作用是先计算表达式`expression` ,如果其值为假（0）,那么它先向stderr打印一条出错信息，然后通过调用abort终止程序运行。

## union 的相关概念

&emsp;&emsp;联合体内部数据共享同一段内存。典型应用，三原色的组合。

```c
#include "stdio.h"
 
typedef struct
{
    unsigned char Red;
    unsigned char Green;
    unsigned char Blue;
}RGB_Typedef;
 
typedef union
{
    RGB_Typedef rgb;
    unsigned int value;
}Pix_Typedef;
 
void main()
{
    Pix_Typedef pix;
    printf("%x\r\n",&pix.rgb);
    printf("%x\r\n",&pix.value);
}
```



# Method Swizzle

https://www.jianshu.com/p/51a53975ad0f

+ 功能上类似继承重写方法。
+ Method Swizzle 指的是改变一个已经存在的selector对应的方法实现。
+ 采用AOP的编程思想。
+ Method Swizzing 利用Runtime特性吧一个方法的实现和另一个方法的实现进行替换。
+ 底层原理即在程序运行时修改Dispatch Table 里面的SEL和IMP的映射关系。

## 优势

+ 继承：修改多，无法保证他人一定继承基类

+ 类别：类别中重写方法会覆盖原来的实现，大多时候，重写一个方法是为了添加一些代码，而不是完全取代它。如果两个类别都实现了相同的方法，只有一个类别的方法会被调用到。

+ AOP优势：减少切面业务的重复编码，减少与其他业务的耦合，把琐碎事务从主业务中分离。

## 实现

- swizzling method的实现原理这里不再说明，相关知识：Objective-C调用方法的原理(包括消息解析与转发)。
- 最常见的：在`+(void)load`中自己实现swizzling method的代码。
- Github 上有一个基于 swizzling method 的开源框架 Aspects：`Aspects`是`NSObject`的Category，任何`NSObject`类要hook实例方法或者类方法，只需调用`Aspects`提供的一个简单方法，就可以实现hook，并且可以安排附加代码的执行时机。

# 

## AOP

AOP编程的原理是在不更改正常的业务处理流程的前提下，通过生成一个动态代理类，从而实现对目标对象嵌入附加的操作。AOP是OOP的一个补充。

AOP最常见的使用场景：日志记录、性能统计、安全控制、事务处理、异常处理、调试等。这些事务琐碎，跟主要业务逻辑无关，在很多地方都有，又很难抽象出来单独的模块。



# Category & Extension

## category

+ 在不修改原类的基础上，为一个类扩展方法。
+ 给系统自带的类扩展方法。

### category 能写什么

+ 只能添加方法，不能新增属性。
+ 通过runtime可以添加属性。

### 注意

+ 分类中可以访问原来类中的@protect和@public修饰的成员变量，私有变量只能通过方法访问。
+ 同名方法优先级： `分类` > `本类` > `父类`
+ 多个分类都有同名方法，调用编译器最后编译的方法。

## 类扩展

+ 分类的一个特例，匿名分类。
+ 为一个类添加一些私有的成员变量和方法。

```objective-c
//.m文件中 
//类扩展
@implementation ViewController()
  
@end
@implementation ViewController
  
@end
```

## 区别

1. 分类原则上只能添加方法，不能添加属性。
2. 扩展可以增加方法，还可以增加属性（private）。
3. 扩展声明的方法没实现，编译器会报警（编译阶段被添加到类中）；分类方法没被实现不会报警，因为是在运行时添加到类中。
4. 扩展不能像分类那样拥有独立的实现部分，也就是说，扩展声明的方法必须依托对应类的实现部分进行实现。

# NSCoding

## NSCoder 

+ 是一个抽象类
+ 不能被实例化，提供一些子类继承的方法。

## NSKeyedUnarchiver

+ 从二进制流读取对象。

## NSKeyedArchiver

+ 把对象写到二进制流中。

## NSCoding

+ 是一个协议，主要有两个方法

  + encodeWithCoder
  + initWithEncoder

  ```objective-c
  // 解档方法(从coder中读取数据，保存到相应的变量中，即反序列化数据；实现NScoding协议)
  - (instancetype)initWithCoder:(NSCoder *)coder;
   
  // 归档调用方法(读取实例变量，并把这些数据写到coder中去，即序列化数据)
  - (void)encodeWithCoder:(NSCoder *)coder;
  ```



# KVO



# KVC 





# HTTPS

![](https://camo.githubusercontent.com/9754c28a61cb7c8c91c043e6b46a773c06171ecc32cbe01645c96cb8e3278b03/687474703a2f2f7777312e73696e61696d672e636e2f6c617267652f303036744e6337396c79316733683373766a6332696a3330693030666b6d79702e6a7067)

