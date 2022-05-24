# Block

[toc]

Block本质就是一个OC对象。

![](https://upload-images.jianshu.io/upload_images/6894675-06163c96bc6700ec.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



## Block 三种类型

### __NSGlobalBlock

未访问auto变量的是全局block

### __NSStackBlock

访问了auto变量的。

### __NSMallocBlock

__NSStackBlock copy后



注意，在ARC环境下， 编译器会根据情况自动将栈上的block copy到堆上：

1. block作为函数返回值时。
2. 被强指针引用的block会自动copy。
3. block作为Cocoa API 方法名含有的UsingBlock的参数时。
4. block作为GCD API的方法参数时。

## Block 原理

block的本质是一个对象。

### Block 内存布局

```objective-c
struct Block_descriptor {
  unsigned long int reserved;  //预留内存大小
  unsigned long int size;     //块大小
  void (*copy)(void *dst, void *src); //指向拷贝函数的指针
  void (*dispose)(void*); //指向释放函数的函数指针
};

struct Block_layout {
  void *isa;   //isa指针
  int flags;   //状态标志位
  int reserved;  //预留内存大小
  void (*invoke)(void *,...); //实现指向块实现的函数指针
  struct Block_descriptor *descriptor;
  
};
```

```objective-c
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FunPtr;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int ivar;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _ivar, int flags=0): ivar(_ivar) {
    impl.isa = &_NSConcreteStackBlock;
  }
}
```

## Block捕获自动变量

### 变量分类

> + auto 自动变量：默认方法的作用域中不加前缀就是自动变量,而且ARC下默认还会加__strong.
> + static静态变量：存放在内存的可读写区， 初始化只在第一次执行时起作用，运行过程中语句块变量保持上一次执行的值。
> + static全局静态变量：放在可读写去，作用域为当前文件。
> + 全局变量：可读写区， 整个程序可用。
> + 另外在OC中它们又分为对象类型和非对象类型.



Block的__main_block_impl_0结构体拷贝了一份自动变量作为结构体的成员变量，内部的和外部的变量并不在同一块内存上。

















## 堆Block



## 栈Block



## 全局Block



## Block 变量捕获



## __Block





## weakSelf



















