---
title: RunLoop源码观察
date: 2016-06-28 14:47:47
tags: 
- iOS
- RunLoop
category:
- iOS
---

<!--
create time: 2016-06-26 11:14:42
Author: tufei

This file is created by Marboo<http://marboo.io> template file $MARBOO_HOME/.media/starts/default.md
本文件由 Marboo<http://marboo.io> 模板文件 $MARBOO_HOME/.media/starts/default.md 创建
-->

源码可以在这里[下载](http://opensource.apple.com/tarballs/CF/)  
文件 `CFRunLoop.c CFRunLoop.h`  
这篇是在看了[ibireme的深入理解RunLoop](http://blog.ibireme.com/2015/05/18/runloop/)后自己对照源码并编写测试的一些记录

## `CFRunLoopMode`结构

```c
typedef struct __CFRunLoopMode *CFRunLoopModeRef;

struct __CFRunLoopMode {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;	/* must have the run loop locked before locking this */
    CFStringRef _name;
    Boolean _stopped;
    char _padding[3];
    CFMutableSetRef _sources0;
    CFMutableSetRef _sources1;
    CFMutableArrayRef _observers;
    CFMutableArrayRef _timers;
    CFMutableDictionaryRef _portToV1SourceMap;
    __CFPortSet _portSet;
    CFIndex _observerMask;
#if USE_DISPATCH_SOURCE_FOR_TIMERS
    dispatch_source_t _timerSource;
    dispatch_queue_t _queue;
    Boolean _timerFired; // set to true by the source when a timer has fired
    Boolean _dispatchTimerArmed;
#endif
#if USE_MK_TIMER_TOO
    mach_port_t _timerPort;
    Boolean _mkTimerArmed;
#endif
#if DEPLOYMENT_TARGET_WINDOWS
    DWORD _msgQMask;
    void (*_msgPump)(void);
#endif
    uint64_t _timerSoftDeadline; /* TSR */
    uint64_t _timerHardDeadline; /* TSR */
};
```

## `CFRunLoop`结构

```c
struct __CFRunLoop {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;			/* locked for accessing mode list */
    __CFPort _wakeUpPort;			// used for CFRunLoopWakeUp 
    Boolean _unused;
    volatile _per_run_data *_perRunData;              // reset for runs of the run loop
    pthread_t _pthread;     //RunLoop对应的线程id
    uint32_t _winthread;
    CFMutableSetRef _commonModes;
    CFMutableSetRef _commonModeItems;
    CFRunLoopModeRef _currentMode;      //当前以哪种mode运行
    CFMutableSetRef _modes;
    struct _block_item *_blocks_head;   //RunLoop运行时在DoBlock时会从head遍历到下面的tail去取出_block_item判断是否执行然后执行
    struct _block_item *_blocks_tail;
    CFAbsoluteTime _runTime;
    CFAbsoluteTime _sleepTime;
    CFTypeRef _counterpart;
};

struct _block_item {
    struct _block_item *_next;
    CFTypeRef _mode;	// CFString or CFSet
    void (^_block)(void);
};
```

## 获取RunLoop
存储\<key:thread,value:runloop\>全局字典默认是没有的，也就是说App初始化时是没有任何RunLoop的，在获取RunLoop时如果没有，就会创建，并存储在这个全局字典里

```c
//获取MainRunLoop
CFRunLoopRef CFRunLoopGetMain(void) {
    CHECK_FOR_FORK();
    static CFRunLoopRef __main = NULL; // no retain needed
    if (!__main) __main = _CFRunLoopGet0(pthread_main_thread_np()); // no CAS needed
    return __main;
}

//获取当前RunLoop
CFRunLoopRef CFRunLoopGetCurrent(void) {
    CHECK_FOR_FORK();
    CFRunLoopRef rl = (CFRunLoopRef)_CFGetTSD(__CFTSDKeyRunLoop);
    if (rl) return rl;
    return _CFRunLoopGet0(pthread_self());
}

CF_EXPORT CFRunLoopRef _CFRunLoopGet0(pthread_t t) {
    if (pthread_equal(t, kNilPthreadT)) {
    	t = pthread_main_thread_np();
    }
    __CFLock(&loopsLock);
    //一个字典，key为线程id，value为对应的RunLoop
    if (!__CFRunLoops) {
        __CFUnlock(&loopsLock);
    	CFMutableDictionaryRef dict = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, &kCFTypeDictionaryValueCallBacks);
    	CFRunLoopRef mainLoop = __CFRunLoopCreate(pthread_main_thread_np());
    	CFDictionarySetValue(dict, pthreadPointer(pthread_main_thread_np()), mainLoop);
    	if (!OSAtomicCompareAndSwapPtrBarrier(NULL, dict, (void * volatile *)&__CFRunLoops)) {
    	    CFRelease(dict);
    	}
    	CFRelease(mainLoop);
        __CFLock(&loopsLock);
    }
    CFRunLoopRef loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
    __CFUnlock(&loopsLock);
    //用线程id去字典中查找RunLoop，如果要获取的RunLoop在字典中没有，则创建一个RunLoop
    if (!loop) {
    	CFRunLoopRef newLoop = __CFRunLoopCreate(t);
        __CFLock(&loopsLock);
    	loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
    	if (!loop) {
    	    CFDictionarySetValue(__CFRunLoops, pthreadPointer(t), newLoop);
    	    loop = newLoop;
    	}
        // don't release run loops inside the loopsLock, because CFRunLoopDeallocate may end up taking it
        __CFUnlock(&loopsLock);
    	CFRelease(newLoop);
    }
    //这里会把当前正在执行的RunLoop放在一个Set里，在获取CurrentRunLoop时会优先在这个Set里查找
    if (pthread_equal(t, pthread_self())) {
        _CFSetTSD(__CFTSDKeyRunLoop, (void *)loop, NULL);
        if (0 == _CFGetTSD(__CFTSDKeyRunLoopCntr)) {
            _CFSetTSD(__CFTSDKeyRunLoopCntr, (void *)(PTHREAD_DESTRUCTOR_ITERATIONS-1), (void (*)(void *))__CFFinalizeRunLoop);
        }
    }
    return loop;
}
```

## 创建`RunLoop`

```c
static CFRunLoopRef __CFRunLoopCreate(pthread_t t) {
    CFRunLoopRef loop = NULL;
    CFRunLoopModeRef rlm;
    uint32_t size = sizeof(struct __CFRunLoop) - sizeof(CFRuntimeBase);
    loop = (CFRunLoopRef)_CFRuntimeCreateInstance(kCFAllocatorSystemDefault, CFRunLoopGetTypeID(), size, NULL);
    if (NULL == loop) {
       return NULL;
   }
   (void)__CFRunLoopPushPerRunData(loop);
   __CFRunLoopLockInit(&loop->_lock);
   loop->_wakeUpPort = __CFPortAllocate();
   if (CFPORT_NULL == loop->_wakeUpPort) HALT;
   __CFRunLoopSetIgnoreWakeUps(loop);
   loop->_commonModes = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
   
    //kCFRunLoopDefaultMode有commonMode标志
   CFSetAddValue(loop->_commonModes, kCFRunLoopDefaultMode);
   loop->_commonModeItems = NULL;
   loop->_currentMode = NULL;
   loop->_modes = CFSetCreateMutable(kCFAllocatorSystemDefault, 0, &kCFTypeSetCallBacks);
   loop->_blocks_head = NULL;
   loop->_blocks_tail = NULL;
   loop->_counterpart = NULL;
   loop->_pthread = t;
#if DEPLOYMENT_TARGET_WINDOWS
   loop->_winthread = GetCurrentThreadId();
#else
   loop->_winthread = 0;
#endif
    //初始会把kCFRunLoopDefaultMode加入到 loop->_modes里面
   rlm = __CFRunLoopFindMode(loop, kCFRunLoopDefaultMode, true);
   if (NULL != rlm) __CFRunLoopModeUnlock(rlm);
   return loop;
}
```




## **modeItem的种类**  
modeName的种类可以在[这里看到更多](http://iphonedevwiki.net/index.php/CFRunLoop)  
文档中关于RunLoop sources的描述
>A run loop receives events from two different types of sources. **Input sources** deliver asynchronous events, usually messages from another thread or from a different application. **Timer sources** deliver synchronous events, occurring at a scheduled time or repeating interval. Both types of source use an application-specific handler routine to process the event when it arrives.

`CFRunLoopSourceRef`是事件源（输入源）  
比如点击事件，触摸事件等  
按照**官方文档**分类，Source可分为：  

1. `Port-Base Source`：基于端口，和其他线程进行交互  
2. `Custom Input Source`：自定义输入源  
3. `Cocoa Perform Selector Source`：performSelector相关函数  

按照**函数调用栈**，Source可分为：  

1. `Source0`：非基于Port的，接收点击事件，触摸事件等等  

  * 比如某个按钮点击的调用栈是添加Source0给RunLoop
    ![](http://7xsw3s.com1.z0.glb.clouddn.com/1466928381188)  
  * `performSelector`没有delay时 是添加source0给RunLoop
    ![](http://7xsw3s.com1.z0.glb.clouddn.com/1466954405760)
  * 但是在 `performSelector`有delay时 是添加TimerSource给RunLoop
    ![](http://7xsw3s.com1.z0.glb.clouddn.com/1466954585509)

2. `Source1`：基于Port的，通过内核和其他线程通信，接收，分发系统事件（大部分事件都是由屏幕表面包装成Event，然后分发下去，由Source0去处理）

[ibireme博客的解释](http://blog.ibireme.com/2015/05/18/runloop/)
Source有两个版本：Source0 和 Source1。  
• Source0 只包含了一个回调（函数指针），它并不能主动触发事件。使用时，你需要先调用 CFRunLoopSourceSignal(source)，将这个 Source 标记为待处理，然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 RunLoop，让其处理这个事件。  
• Source1 包含了一个 mach_port 和一个回调（函数指针），被用于通过内核和其他线程相互发送消息。这种 Source 能主动唤醒 RunLoop 的线程，其原理在下面会讲到。  

关于`mach port`的介绍可以查看[这里(进程间通信)](https://segmentfault.com/a/1190000002400329)

## 各种modeItem的结构
```c
struct __CFRunLoopSource {
    CFRuntimeBase _base;
    uint32_t _bits;
    pthread_mutex_t _lock;
    CFIndex _order;			/* immutable */
    CFMutableBagRef _runLoops;
    //source分source0,source1
    union {
    	CFRunLoopSourceContext version0;	/* immutable, except invalidation */
        CFRunLoopSourceContext1 version1;	/* immutable, except invalidation */
    } _context;
};

struct __CFRunLoopObserver {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;
    CFIndex _rlCount;
    CFOptionFlags _activities;		/* immutable */
    CFIndex _order;			/* immutable */
    CFRunLoopObserverCallBack _callout;	/* immutable */
    CFRunLoopObserverContext _context;	/* immutable, except invalidation */
};

struct __CFRunLoopTimer {
    CFRuntimeBase _base;
    uint16_t _bits;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;
    CFMutableSetRef _rlModes;
    CFAbsoluteTime _nextFireDate;
    CFTimeInterval _interval;		/* immutable */
    CFTimeInterval _tolerance;          /* mutable */
    uint64_t _fireTSR;			/* TSR units */
    CFIndex _order;			/* immutable */
    CFRunLoopTimerCallBack _callout;	/* immutable */
    CFRunLoopTimerContext _context;	/* immutable, except invalidation */
};
```

## RunLoop运行
文档中的描述

> 1. Notify observers that the run loop has been entered.
2. Notify observers that any ready timers are about to fire.
3. Notify observers that any input sources that are not port based are about to fire.
4. Fire any non-port-based input sources that are ready to fire.
5. If a port-based input source is ready and waiting to fire, process the event immediately. Go to step 9.
6. Notify observers that the thread is about to sleep.
7. Put the thread to sleep until one of the following events occurs:
  1. An event arrives for a port-based input source.
  2. A timer fires.
  3. The timeout value set for the run loop expires.
  4. The run loop is explicitly woken up.
8. Notify observers that the thread just woke up.
9. Process the pending event.
  1. If a user-defined timer fired, process the timer event and restart the loop. Go to step 2.
  2. If an input source fired, deliver the event.
  3. If the run loop was explicitly woken up but has not yet timed out, restart the loop. Go to step 2.
10. Notify observers that the run loop has exited.

翻译：

1. 通知Observers准备进入RunLoop了
2. 通知Observers准备执行Timers了
3. 通知OBservers准备执行非Port-Base的inputSource了
4. 执行非Port-Base的inputSource
5. 如果有Port-Base的事件，并且等待执行，跳到第9步
6. 通知Observers RunLoop没事情做了，准备休眠了（挂起）
7. RunLoop休眠，直到下面任何一个事情发生
  1. 产生了一个Port-Base的source
  2. 定时器时间到了
  3. RunLoop设置的执行时长到了
  4. RunLoop被显式唤起
8. 通知Observers RunLoop要被唤醒了
9. 处理事件
  1. 如果一个用户自定义的Timer定时器到了，执行定时器指定的方法，并重启RunLoop循环（restart RunLoop 就是继续执行RunLoop 的do-while循环），跳到第2步
  2. 如果是一个InputSource，则派发事件（比如屏幕点击，触摸等等）
  3. 如果是被显式唤醒而且没有超过RunLoop指定的执行时长，则重启RunLoop循环
10. 通知Observers RunLoop要退出了

**注意**：进入这里循环前，首先必须是当前RunLoop在指定Mode下，是有modeItems的，如果没有，会直接退出RunLoop，不会进入主循环

```c
void CFRunLoopRun(void) {	/* DOES CALLOUT */
    int32_t result;
    do {
        result = CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
        CHECK_FOR_FORK();
    } while (kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result);
}

//指定ModeName，执行时长去运行RunLoop
SInt32 CFRunLoopRunInMode(CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    CHECK_FOR_FORK();
    return CFRunLoopRunSpecific(CFRunLoopGetCurrent(), modeName, seconds, returnAfterSourceHandled);
}

SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    CHECK_FOR_FORK();
    if (__CFRunLoopIsDeallocating(rl)) return kCFRunLoopRunFinished;
    __CFRunLoopLock(rl);
    
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, modeName, false);
    //如果当前RunLoop没有这个modeName，或者当前modeName中没有modeItem则退出RunLoop
    if (NULL == currentMode || __CFRunLoopModeIsEmpty(rl, currentMode, rl->_currentMode)) {
    	Boolean did = false;
    	if (currentMode) __CFRunLoopModeUnlock(currentMode);
    	__CFRunLoopUnlock(rl);
    	return did ? kCFRunLoopRunHandledSource : kCFRunLoopRunFinished;
    }
    volatile _per_run_data *previousPerRun = __CFRunLoopPushPerRunData(rl);
    //保存之前的执行mode，在当前mode执行完退出RunLoop后，会使用之前的mode继续运行RunLoop
    CFRunLoopModeRef previousMode = rl->_currentMode;
    rl->_currentMode = currentMode;
    int32_t result = kCFRunLoopRunFinished;

    //1. 通知Observers进入RunLoop主循环
    if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
    //RunLoop主循环
    result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
    //10. 通知Observers退出RunLoop主循环
    if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);

    __CFRunLoopModeUnlock(currentMode);
    __CFRunLoopPopPerRunData(rl, previousPerRun);
    //RunLoop在当前mode执行完退出后，mode切换为之前的mode
    //比如我们在滑动scrollView时，MainRunLoop先是在kCFRunLoopDefaultMode启
    //动，滑动时先退出当前mode，然后以UITrackingRunLoopMode进入RunLoop，当滑
    //动结束时，切换回kCFRunLoopDefaultMode
    rl->_currentMode = previousMode;
    __CFRunLoopUnlock(rl);
    return result;
}

//RunLoop主循环
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {

    Boolean didDispatchPortLastTime = true;
    int32_t retVal = 0;
    do {
        
        //2. 通知Observers准备执行Timers了
        if (rlm->_observerMask & kCFRunLoopBeforeTimers) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        //3. 通知OBservers准备执行非Port-Base的inputSource了
        if (rlm->_observerMask & kCFRunLoopBeforeSources) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);
        //执行Block
        __CFRunLoopDoBlocks(rl, rlm);
        
        //4. 执行非Port-Base的inputSource
        Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
        if (sourceHandledThisLoop) {
            __CFRunLoopDoBlocks(rl, rlm);
        }

        Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);

        //5. 如果有Port-Base的事件，并且等待执行，跳到第9步
        //调用 mach_msg 等待接受 mach_port 的消息。线程将进入休眠, 直到被下面某一个事件唤醒。
            /// • 一个基于 port 的Source 的事件。
            /// • 一个 Timer 到时间了
            /// • RunLoop 自身的超时时间到了
            /// • 被其他什么调用者手动唤醒
        if (MACH_PORT_NULL != dispatchPort && !didDispatchPortLastTime) {
            msg = (mach_msg_header_t *)msg_buffer;
            if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0, &voucherState, NULL)) {
                goto handle_msg;
            }
        }

        didDispatchPortLastTime = false;

        //6. 通知Observer，即将进入休眠
        if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting))
             __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
        //7. RunLoop进入休眠
        __CFRunLoopSetSleeping(rl);
        
        //8. 通知Observers，RunLoop被唤醒
        if (!poll && (rlm->_observerMask & kCFRunLoopAfterWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);
        
        //9. 收到消息，处理消息。
        handle_msg:;
            /// 9.1 如果一个 Timer 到时间了，触发这个Timer的回调。
            if (msg_is_timer) {
                __CFRunLoopDoTimers(runloop, currentMode, mach_absolute_time())
            } 
 
            /// 9.2 如果有dispatch到main_queue的block，执行block。
            else if (msg_is_dispatch) {
                __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
            } 
 
            /// 9.3 如果一个 Source1 (基于port) 发出事件了，处理这个事件
            else {
                CFRunLoopSourceRef source1 = __CFRunLoopModeFindSourceForMachPort(runloop, currentMode, livePort);
                sourceHandledThisLoop = __CFRunLoopDoSource1(runloop, currentMode, source1, msg);
                if (sourceHandledThisLoop) {
                    mach_msg(reply, MACH_SEND_MSG, reply);
                }
            }
          
            __CFRunLoopDoBlocks(rl, rlm);
              

            if (sourceHandledThisLoop && stopAfterHandle) {
                // 进入loop时参数说处理完事件就返回。
                retVal = kCFRunLoopRunHandledSource;
            } else if (timeout_context->termTSR < mach_absolute_time()) {
                // 超出传入参数标记的超时时间了
                retVal = kCFRunLoopRunTimedOut;
            } else if (__CFRunLoopIsStopped(rl)) {
                // 被外部调用者强制停止了
                __CFRunLoopUnsetStopped(rl);
                retVal = kCFRunLoopRunStopped;
            } else if (rlm->_stopped) {
                rlm->_stopped = false;
                retVal = kCFRunLoopRunStopped;
            } else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode)) {
                // source/timer/observer一个都没有了
                retVal = kCFRunLoopRunFinished;
            }

    } while (0 == retVal);

    return retVal;
}

```
