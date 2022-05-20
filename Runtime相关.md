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



# Method Swizzle



# Category & Extension



# NSCoding



# KVO



# KVC 

