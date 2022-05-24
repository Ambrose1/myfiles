# [RunLoop][https://blog.csdn.net/qq_45836906/article/details/119578794]

[toc]

## 什么是RunLoop

保持线程随时处理事件不退出的机制。

+ 无事件时，runloop进入休眠状态。
+ 有事件时， 处理事件。

### 基本作用

1. 保持程序持续运行
2. 处理App各种事件
3. 节省CPU资源，提高程序性能。

### 获取runloop

+ Foundation NSRunLoop
+ Core Foundation CFRunLoopRef

Apple不允许直接创建RunLoop， 可以通过下列方法获取runloop对象。

```objective-c
    //获取主线程的runloop
    NSRunLoop *runloop1 = [NSRunLoop mainRunLoop];
    CFRunLoopRef runloop2 = CFRunLoopGetMain();
    
    //获取当前线程的runloop对象
    NSRunLoop *runloop3 = [NSRunLoop currentRunLoop];
    CFRunLoopRef runloop4 = CFRunLoopGetCurrent();

```

+ 线程与runloop是一一对应的，保存在一个全局字典中，线程作为key， runloop是value
+ 线程刚创建的时候没有runloop， 需要主动获取。
+ runloop的创建是发生在第一次获取时，销毁在线程结束时。
+ 主线程的runloop在程序启动时就启动（在main.m函数中，通过UIApplicationMain开启主线程的runloop）

###  runloop结构

```objective-c
struct __CFRunLoop {
  CFRuntimeBase __base;
  pthread_mutex_t _lock;   //访问mode的锁
  __CFPort _wakeUpPort;
  Boolean _unused;
  volatile _per_run_data *_perRunData;
  
  pthread_t _pthread;  //runloop 对应的线程
  unit32_t _winthread;
  
  CFMutableSetRef _commonModes;     //标记为common的mode
  CFMutableSetRef _commonModeItems;   //存储CommonMode 
  CFRunLoopModeRef _currentMode;
  CFMutableSetRef _modes;
  
  struct _block_item *_blocks_head;
  struct _block_item *_blocks_tail;
  CFAbsoluteTime _runTime;
  CFAbsoluteTime _sleepTime;
  CFTypeRef _counterpart;
  
};
```

![](https://img-blog.csdnimg.cn/a73db7e073124219837fbe65f1d4d883.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1ODM2OTA2,size_16,color_FFFFFF,t_70)

一个runloop对象中，主要包含一个线程，若干个mode，若干个commonMode, 还有一个当前运行的mode。

+ CFRunLoopModeRef  runloop的运行模式
+ 一个RunLoop包含若干个Mode，每一个Mode又包含若干个Source0/Source1/Timer/Observer（下面有Mode的结构）
+ RunLoop启动时只能选择一个Mode 作为currentMode
+ 切换mode需要退出当前Loop，重新选择一个Mode进入，Mode 切换不会导致程序退出。
+ 不同mode的Source0、Source1,Timer,Observer能分割开，互不影响。
+ Model中如果没有任何Source0/Source1，Timer，Observer, RunLoop会立刻退出。

> CFRunLoopRef：代表了RunLoop的对象（RunLoop）
> CFRunLoopModeRef：RunLoop的运行模式（Mode）
> CFRunLoopSourceRef：RunLoop模型图中的输入源/事件源（Source）
> CFRunLoopTimerRef：RunLoop模型图中的定时源（Timer）
> CFRunLoopObserverRef：观察者，能够监听RunLoop的状态变化
>
> 原文链接：https://blog.csdn.net/qq_45836906/article/details/119578794

## RunLoop Mode

### 结构简化

```objective-c
struct __CFRunLoopMode {
    CFMutableSetRef _sources0;
    CFMutableSetRef _sources1;
    CFMutableArrayRef _observers;
    CFMutableArrayRef _timers;
};
```

Mode实际上是Source，Timer 和 Observer 的集合，不同的Mode把不同组的Source、timer、Observer隔绝开来。runloop在某一时刻只能运行在一个mode下，处理这一个mode中的source、timer、observer。

### 五种Mode

#### kCFDefaultRunLoopMode

App默认mode， 通常主线程在这个mode下运行。

#### UITrackingRunLoopMode

页面追踪mode， 用于ScrollView追踪触摸滑动，保证界面滑动时不受其他Mode影响。

#### UIInitializationRunLoopMode

刚启动App时的第一个Mode, 启动完成后就不再使用，会切换到kCFRunLoopDefaultMode

#### GSEventReceiveRunLoopMode

接受系统事件内部Mode， 通常用不到

#### kCFRunLoopCommonModes

是一种标记



> RunLoop中包含多个Mode, 只有一个作为当前Mode

Mode 中的Source/Observer/Timer三种统称为ModeItem.

### Modeltem

#### CFRunLoopSourceRef

CFRunLoopSource 包含Source0和Source1两个版本。

```objective-c

struct __CFRunLoopSource {
    CFRuntimeBase _base;
    uint32_t _bits; 	//用于标记Signaled状态，source0只有在被标记为Signaled状态才会被处理
    pthread_mutex_t _lock;
    CFIndex _order;			/* immutable */
    CFMutableBagRef _runLoops;
    union {
	CFRunLoopSourceContext version0;	/* immutable, except invalidation */
        CFRunLoopSourceContext1 version1;	/* immutable, except invalidation */
    } _context;
};

```

Source0 是App内部事件，由App自己管理的UIEvent、CFSocket都是source0.当一个Source0事件准备执行时，必须要先把它标记为signal状态。

```objective-c
typedef struct {
    CFIndex	version;
    void *	info;
    const void *(*retain)(const void *info);
    void	(*release)(const void *info);
    CFStringRef	(*copyDescription)(const void *info);
    Boolean	(*equal)(const void *info1, const void *info2);
    CFHashCode	(*hash)(const void *info);
    void	(*schedule)(void *info, CFRunLoopRef rl, CFRunLoopMode mode);
    void	(*cancel)(void *info, CFRunLoopRef rl, CFRunLoopMode mode);
    void	(*perform)(void *info);
} CFRunLoopSourceContext;

```

Source1 结构体

```objective-c
typedef struct {
    CFIndex	version;
    void *	info;
    const void *(*retain)(const void *info);
    void	(*release)(const void *info);
    CFStringRef	(*copyDescription)(const void *info);
    Boolean	(*equal)(const void *info1, const void *info2);
    CFHashCode	(*hash)(const void *info);
#if TARGET_OS_OSX || TARGET_OS_IPHONE
    mach_port_t	(*getPort)(void *info);
    void *	(*perform)(void *msg, CFIndex size, CFAllocatorRef allocator, void *info);
#else
    void *	(*getPort)(void *info);
    void	(*perform)(void *info);
#endif
} CFRunLoopSourceContext1;

```

##### 区别

**Source0:** 

1. 非基于port。
2. 只包含一个回调，不能主动触发事件。
3. 使用时需要先调用CFRunLoopSourceSignal, 将这个source标记为待处理，然后手动调用CFRunLoopWakeUp唤醒RunLoop，让其处理这个事件。

**Source1**

1. 包含mach_port和一个回调。
2. 可以监听系统端口，通过内核和其他线程通信、接收、分发系统事件。
3. 能主动唤醒RunLoop.



#### CFRunLoopTimerRef

```objective-c
struct __CFRunLoopTimer {
    CFRuntimeBase _base;
    uint16_t _bits;                 //标记fire状态
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;      //添加该Timer的runloop
    CFMutableSetRef _rlModes;   //存放所有 包含该timer的 mode的 modeName，意味着一个timer可以在多个mode中存在
    CFAbsoluteTime _nextFireDate;
    CFTimeInterval _interval;	 //理想时间间隔	/* immutable */
    CFTimeInterval _tolerance;   //时间偏差       /* mutable */
    uint64_t _fireTSR;			/* TSR units */
    CFIndex _order;			/* immutable */
    CFRunLoopTimerCallBack _callout;	/* immutable */
    CFRunLoopTimerContext _context;	/* immutable, except invalidation */
};

```

CFRunLoopTimer是基于时间的触发器，其包含一个时间长度、一个回调。当其加入runloop时，runloop会注册对应的时间点，并定时唤醒runloop执行那个回调。

CFRunLoopTimer和NSTimer是toll-free bridged对象桥接，可以相互转换。

#### CFRunLoopObserveRef

```objective-c
struct __CFRunLoopObserver {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;   //添加了该observer的runloop
    CFIndex _rlCount;
    CFOptionFlags _activities;		/* immutable */
    CFIndex _order;			/* immutable */
    CFRunLoopObserverCallBack _callout;	/* immutable */  //设置回调函数，回调指针
    CFRunLoopObserverContext _context;	/* immutable, except invalidation */
};

```

CFRunLoopObserverRef 是观察者，可以观察RunLoop的各种状态，每个Observer都包含一个回调，当runloop的状态发生变化是，观察者就能通过回到接受这个变化。

##### RunLoop的六种状态

```objective-c
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),         //即将进入Loop
    kCFRunLoopBeforeTimers = (1UL << 1),  //即将处理Timer
    kCFRunLoopBeforeSources = (1UL << 2), //即将处理Source
    kCFRunLoopBeforeWaiting = (1UL << 5), //即将进入休眠
    kCFRunLoopAfterWaiting = (1UL << 6),  //刚从休眠中唤醒但还没开始处理事件
    kCFRunLoopExit = (1UL << 7),		  //即将退出Loop
    kCFRunLoopAllActivities = 0x0FFFFFFFU //所有状态
};

```



以上的Source、Timer、Observer被统称为mode item，一个item可以被同时加入多个Mode中，但一个item被重复加入同一个mode时是不会有效果的。如果一个mode中一个item都没有，则runloop会直接退出。

## RunLoop 运行逻辑

![](https://img-blog.csdnimg.cn/e2bf1432995c4d4c905bc67ff29eb15a.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1ODM2OTA2,size_16,color_FFFFFF,t_70)

```objective-c
//用DefaultMode启动
void CFRunLoopRun(void) {
    CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
}

//用指定的Mode启动，允许设置RunLoop的超时时间
int CFRunLoopRunInMode(CFStringRef modeName, CFTimeInterval seconds, Boolean stopAfterHandle) {
    return CFRunLoopRunSpecific(CFRunLoopGetCurrent(), modeName, seconds, returnAfterSourceHandled);
}

//RunLoop的实现
int CFRunLoopRunSpecific(runloop, modeName, seconds, stopAfterHandle) {
	//首先根据modeName找到对应的mode
	CFRunLoopModeRef currentMode = __CFRunLoopFindMode(runloop, modeName, false);
	//如果mode中没有source/timer/observer，直接返回
	if (__CFRunLoopModeIsEmpty(currentMode)) return;
	
	//1.通知Observers：RunLoop即将进入loop
	__CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopEntry);

	//调用函数__CFRunLoopRun 进入loop
	__CFRunLoopRun(runloop, currentMode, seconds, returnAfterSourceHandled) {
        
        Boolean sourceHandledThisLoop = NO;
        int retVal = 0;
        do {
			//2.通知Observers：RunLoop即将触发Timer回调
			__CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeTimers);
			//3.通知Observers：RunLoop即将触发Source0（非port）回调
			__CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeSources);
			///执行被加入的block
			__CFRunLoopDoBlocks(runloop, currentMode);

			//4.RunLoop触发Source0（非port）回调
			sourceHandledThisLoop = __CFRunLoopDoSources0(runloop, currentMode, stopAfterHandle);
			//执行被加入的Block
			__CFRunLoopDoBlocks(runloop, currentMode);

			//5.如果有Source1（基于port）处于ready状态，直接处理这个Source1然后跳转去处理消息
			if (__Source0DidDispatchPortLastTime) {
                Boolean hasMsg = __CFRunLoopServiceMachPort(dispatchPort, &msg)
                if (hasMsg) goto handle_msg;
            }

			//6.通知Observers：RunLoop的线程即将进入休眠
			if (!sourceHandledThisLoop) {
                __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeWaiting);
            }

			//7.调用mach_msg等待接收mach_port的消息。线程将进入休眠，直到被下面某个事件唤醒
			// 一个基于port的Source的事件
			//一个Timer时间到了
			//RunLoop自身的超时时间到了
			//被其他什么调用者手动唤醒
			__CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort) {
                mach_msg(msg, MACH_RCV_MSG, port); // thread wait for receive msg
            }

			//8.通知Observers：RunLoop的线程刚刚被唤醒
			__CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopAfterWaiting);

			//收到消息，处理消息
			handle_msg:
			//9.1 如果一个Timer时间到了，触发这个timer的回调
			if (msg_is_timer) {
                __CFRunLoopDoTimers(runloop, currentMode, mach_absolute_time())
            } 
			//9.2 如果有dispatch到main_queue的block，执行block
			else if (msg_is_dispatch) {
                __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
            } 
            //9.3 如果一个Source1（基于port）发出事件了，处理这个事件
            else {
                CFRunLoopSourceRef source1 = __CFRunLoopModeFindSourceForMachPort(runloop, currentMode, livePort);
                sourceHandledThisLoop = __CFRunLoopDoSource1(runloop, currentMode, source1, msg);
                if (sourceHandledThisLoop) {
                    mach_msg(reply, MACH_SEND_MSG, reply);
                }
            }
            //执行加入到loop的block
            __CFRunLoopDoBlocks(runloop, currentMode);


			if (sourceHandledThisLoop && stopAfterHandle) {
                /// 进入loop时参数说处理完事件就返回。
                retVal = kCFRunLoopRunHandledSource;
            } else if (timeout) {
                /// 超出传入参数标记的超时时间了
                retVal = kCFRunLoopRunTimedOut;
            } else if (__CFRunLoopIsStopped(runloop)) {
                /// 被外部调用者强制停止了
                retVal = kCFRunLoopRunStopped;
            } else if (__CFRunLoopModeIsEmpty(runloop, currentMode)) {
                /// source/timer/observer一个都没有了
                retVal = kCFRunLoopRunFinished;
            }

			// 如果没超时，mode里没空，loop也没被停止，那继续loop。
	 	} while (retVal == 0);
    }

	//10. 通知Observers：RunLoop即将退出
	__CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
}

```

> 实际上RunLoop就是这样一个函数，其内部就是一个do-while循环，当调用CF RunLoopRun()时，线程就会一直停留在这个循环里，直到超时或被手动停止，该函数才会被返回。

#### RunLoop 回调

1. App启动时，系统会默认注册5个mode

2. 当RunLoop进行回调时，一般都是通过一个很长的函数调出去（call out），当在代码中加断点调试时，通常能在调用栈上看到这些函数。这就是runloop的流程：

   ```objective-c
   {
   	//	1. 通知Observers，即将进入runloop
   	//此处有Observer会创建AutoreleasePool：_objc_autoreleasePoolPush()
   	__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopEntry);
   	do {
   		//2. 通知Observers：即将触发Timer回调
   		__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeTimers);
   		//3. 通知Observers:即将触发Source0回调
   		__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeSources);
           __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
   		
   		//4.触发Source0回调
   		__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(source0);
           __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
   		
   		//6. 通知Observers，即将进入休眠
   		//此处Observer释放并新建autoreleasePool：_objc_autoreleasePoolPop(); _objc_autoreleasePoolPush();
   		__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeWaiting);
   
   		//7. 休眠，等待唤醒
   		mach_msg() -> mach_msg_trap();
   
   		//8. 通知Observers，线程被唤醒
   		__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopAfterWaiting);
   
   		//9. 如果是Timer唤醒的，回调Timer
   		__CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(timer);
   		//9. 如果是被dispatch唤醒的，执行所有调用dispatch_async等方法放入main queue 的block
   		__CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(dispatched_block);
   		//9.如果runloop是被Source1（基于port）的事件唤醒，处理这个事件
   		__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__(source1);
   
   
   	} while (...);
   
   	//10. 通知Observers，即将退出runloop
   	//此处有Observer释放AutoreleasePool：_objc_autoreleasePoolPop();
   	__CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopExit);
   
     
   ```



![](https://img-blog.csdnimg.cn/94663018790649faa7d5f5a0e4c93eb8.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ1ODM2OTA2,size_16,color_FFFFFF,t_70)

## RunLoop在实际开发中的应用

1. 控制线程生命周期（保活）
2. 解决NSTimer在滑动时停止工作的问题
3. 监控应用卡顿
4. 性能优化



















## 如何实现常驻线程

