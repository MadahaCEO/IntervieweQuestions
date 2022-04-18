#RunLoop

RunLoop又叫运行循环，内部就是一个do while的循环，在这个循环内部不断处理各种任务，它在循环监听着各种事件源、消息，对他们进行管理并分发给线程来执行，保证程序持续运行。



##保持程序持续运行。
App一启动就会开启主线程，主线程在开启的时候就会开启主线程对应的RunLoop，能保证线程不被销毁，主线程不销毁，程序就会持续运行。

```
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```
UIApplicationMain函数内部启动了runloop，因此UIApplicationMain函数是一直都没有返回的，保持了程序的持续运行。而这个默认启动的runloop是和主线程相关的

##处理App中各类事件。
事件响应、手势识别、界面刷新、AutoreleasePool自动释放池、NSTimer等事件处理。

##节省CPU资源，提高程序性能。
RunLoop存在的目的就是当线程中有任务的时候，保证线程干活，当线程没有任务的时候，让线程睡眠，提高程序性能，节省资源，



##RunLoop和CFRunLoopRef
iOS中有两套API来访问和使用RunLoop：Foundation中的NSRunLoop、Core Foundation中的CFRunLoopRef，其中的NSRunLoop和CFRunLoopRef都代表着RunLoop对象，NSRunLoop是基于CFRunLoopRef的一层OC包装
CFRunLoop是开源的，地址是：https://opensource.apple.com/tarballs/CF/，CFRunLoopRef提供了两个自动获取RunLoop的函数：CFRunLoopGetMain()、CFRunLoopGetCurrent()，其内部逻辑如下：


##RunLoop和线程之间的关系
* RunLoop保存在一个全局的Dictionary里面，线程为key，RunLoop为Value。
* 线程刚创建的时候是没有RunLoop对象的，RunLoop会在第一次获取它的时候创建。
* RunLoop会在线程结束的时候销毁【解释上面为什么子线程延迟方法没有调用，线程执行很快，瞬间就结束啦，没有runloop为其保活】。
* 主线程的RunLoop已经自动获取（创建），子线程默认没有开启RunLoop。
* 每条线程都有唯一的一个与之对应的RunLoop对象。
* 先有线程，再有RunLoop

runloop在第一次获取时创建，然后在线程结束时销毁。所以，在子线程如果不手动获取runloop，它是一直都不会有的。
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

if (!loop) {
/// 取不到时，创建一个
loop = _CFRunLoopCreate();
CFDictionarySetValue(loopsDic, thread, loop);
/// 注册一个回调，当线程销毁时，顺便也销毁其对应的 RunLoop。
_CFSetTSD(..., thread, loop, __CFFinalizeRunLoop);
}

```



```

- (void)test {

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


```

- (void)test {

    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        [self delayToDoOne];
        [self performSelector:@selector(delayToDoTwo) withObject:nil afterDelay:.5];
        
        // runloop 代码下载下面，不能写在上面。runloog的本质就是持续循环，那么代码肯定就不会向下走，所以，上面的方法肯定不会继续执行了
         NSRunLoop *rp = [NSRunLoop currentRunLoop];
        [rp addPort:[NSMachPort port] forMode:NSRunLoopCommonModes];
        [rp run];
    });
    
    return YES;
}

2022-04-12 20:37:16.194352+0800 CE[1155:58208] delayToDoTwo

```

##Mode类型

* NSDefaultRunLoopMode: 应用程序的默认Mode，通常主线程是在这个Mode下运行。
* NSRunLoopCommonModes: 这是一个占位用的Mode，作为标记KCFRunLoopDefaultMode和UITrackingRunLoopMode用，对几个模式（Mode）进行标记的一个集合, 并不是一种真正的Mode。`但是某一个时刻只能有一个确定的Mode，就是currentMode.【重要线索】`
* UITrackingRunLoopMode: 界面跟踪Mode，用于ScrollView追踪触摸滑动，保证界面滑动的时候不受其他Mode影响。

##解决NSTimer在滑动时停止工作的问题（将Timer添加到CommonMode里面即可）。

```
NSTimer *timer = [NSTimer timerWithTimeInterval:1 target:self selector:@selector(timerEvent) userInfo:nil repeats:YES];
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];

```
Timer默认是处在NSDefaultRunLoopMode模式，当我们滑动页面的时候RunLoop会切换到UITrackingRunLoopMode模式，这样我们的timer就停止工作了，进而导致timer不准确。NSRunLoopCommonModes占位用的mode，它不是真正意义上的mode。这个模式等效于NSDefaultRunLoopMode和UITrackingRunLoopMode的结合。所以给timer指定NSRunLoopCommonModes模式，这样 就可以在NSDefaultRunLoopMode、UITrackingRunLoopMode模式下都运行。


##RunLoop 和 autoreleasepool的关系


##另外可以通过监测RunLoop的状态监测应用卡顿。
```
/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),  // 即将进入loop
    kCFRunLoopBeforeTimers = (1UL << 1),  // 即将处理timer
    kCFRunLoopBeforeSources = (1UL << 2), // 即将处理 source
    kCFRunLoopBeforeWaiting = (1UL << 5),   // 即将进入休眠
    kCFRunLoopAfterWaiting = (1UL << 6),   // 即将从休眠中被唤醒
    kCFRunLoopExit = (1UL << 7),   // 即将推出runloop
    kCFRunLoopAllActivities = 0x0FFFFFFFU  // 默认所有状态
};
```

创建一个观察者，将创建好的观察者添加到主线程的commonMode模式下观察，创建一个持续的子线程专门用来监控主线程的runloop状态，一旦发现进入睡眠前的状态kCFRunLoopBeforeSources，或者唤醒后的状态kCFRunLoopAfterWaiting，在设置的时间阈值内一直没有变化，即可判断为卡顿，dump出堆栈的信息，从而进一步分析出具体是哪个方法的执行时间长。



https://blog.csdn.net/u013378438/article/details/80239686
https://blog.ibireme.com/2015/05/18/runloop/
https://www.jianshu.com/p/14f0d8ea7acd
https://www.jianshu.com/p/2ba1a946ef95
https://www.jianshu.com/p/911549ae4bf8
