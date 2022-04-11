#RunLoop

RunLoop又叫运行循环，内部就是一个do while的循环，在这个循环内部不断处理各种任务，
保证程序持续运行。

##保持程序持续运行。
App一启动就会开启主线程，主线程在开启的时候就会开启主线程对应的RunLoop，能保证线程不被销毁，主线程不销毁，程序就会持续运行。

##处理App中各类事件。
事件响应、手势识别、界面刷新、AutoreleasePool自动释放池、NSTimer等事件处理。

##节省CPU资源，提高程序性能。
RunLoop存在的目的就是当线程中有任务的时候，保证线程干活，当线程没有任务的时候，让线程睡眠，提高程序性能，节省资源，



##RunLoop和CFRunLoopRef
iOS中有两套API来访问和使用RunLoop：Foundation中的NSRunLoop、Core Foundation中的CFRunLoopRef，其中的NSRunLoop和CFRunLoopRef都代表着RunLoop对象，NSRunLoop是基于CFRunLoopRef的一层OC包装
CFRunLoop是开源的，地址是：https://opensource.apple.com/tarballs/CF/，CFRunLoopRef提供了两个自动获取RunLoop的函数：CFRunLoopGetMain()、CFRunLoopGetCurrent()，其内部逻辑如下：



```

/// 全局的Dictionary，key 是 pthread_t， value 是 CFRunLoopRef
static CFMutableDictionaryRef loopsDic;
/// 访问 loopsDic 时的锁
static CFSpinLock_t loopsLock;

/// 获取一个 pthread 对应的 RunLoop。
CFRunLoopRef _CFRunLoopGet(pthread_t thread) {
OSSpinLockLock(&loopsLock);

if (!loopsDic) {
// 第一次进入时，初始化全局Dic，并先为主线程创建一个 RunLoop。
loopsDic = CFDictionaryCreateMutable();
CFRunLoopRef mainLoop = _CFRunLoopCreate();
CFDictionarySetValue(loopsDic, pthread_main_thread_np(), mainLoop);
}

/// 直接从 Dictionary 里获取。
CFRunLoopRef loop = CFDictionaryGetValue(loopsDic, thread));

if (!loop) {
/// 取不到时，创建一个
loop = _CFRunLoopCreate();
CFDictionarySetValue(loopsDic, thread, loop);
/// 注册一个回调，当线程销毁时，顺便也销毁其对应的 RunLoop。
_CFSetTSD(..., thread, loop, __CFFinalizeRunLoop);
}

OSSpinLockUnLock(&loopsLock);
return loop;
}

CFRunLoopRef CFRunLoopGetMain() {
return _CFRunLoopGet(pthread_main_thread_np());
}

CFRunLoopRef CFRunLoopGetCurrent() {
return _CFRunLoopGet(pthread_self());
}


```


每条线程都有唯一的一个与之对应的RunLoop对象

RunLoop保存在一个全局的Dictionary里，线程作为Key，RunLoop作为Value

```
if (!loopsDic) {
// 第一次进入时，初始化全局Dic，并先为主线程创建一个 RunLoop。
loopsDic = CFDictionaryCreateMutable();
CFRunLoopRef mainLoop = _CFRunLoopCreate();
CFDictionarySetValue(loopsDic, pthread_main_thread_np(), mainLoop);
}

/// 直接从 Dictionary 里获取。
CFRunLoopRef loop = CFDictionaryGetValue(loopsDic, thread));

```

线程刚创建时并没有RunLoop对象，RunLoop会在第一次获取它时创建

RunLoop会在线程结束时销毁

主线程的RunLoop默认已经自动创建了，而子线程默认没有开启RunLoop


```

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {

    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        [self delayToDoOne];
        [self performSelector:@selector(delayToDoTwo) withObject:nil afterDelay:.5];
    });
    
    return YES;
}

- (void)delayToDoOne {
    NSLog(@"delayToDoOne");
}

- (void)delayToDoTwo {
    NSLog(@"delayToDoTwo");
}

```
控制台打印如下

```
2022-04-11 20:41:11.712249+0800 CE[1840:91703] delayToDo

```



https://blog.csdn.net/u013378438/article/details/80239686
https://blog.ibireme.com/2015/05/18/runloop/
https://www.jianshu.com/p/14f0d8ea7acd
https://www.jianshu.com/p/2ba1a946ef95
