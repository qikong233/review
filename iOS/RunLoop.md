### RunLoop

#### 基本原理

一般来讲，一个线程只能执行一个任务，执行完成后就会退出。如果我们需要一个机制，让线程随时处理时间但并不退出，通过的逻辑代码是这样的:
```objc
function loop() {
	initialize();
	do {
		var message = get_next_message();
		process_message(message);
	} white(message != quit);
}
```

在iOS中，RunLoop与贤臣是一一对应的关系，这种关系存在一张全局字典中：
```objc
// 全局的Dictionary， key是pthread_t value是CFRunLoopRef
static CFMutableDictionaryRef loopsDic;
// 访问loopsDic时的锁
static CFSpinLock_t loopsLock;

// 获取一个pthread对应的RunLoop
CFRunLoopRef _CFRunLoopGet(pthread_t thread) {
	OSSpinLockLock(&loopsLock);
	if (!loopsDic) {
		//第一次进入，初始化全局Dic，并为主线程创建一个RunLoop
		loopsDic = CFDictionaryCreateMuatble();
		CFRunLoopRef mainLoop = _CFRunLoopCreate();
		CFDictionartSetValue(loopsDic, pthread_main_thread_np(), mainLoop);
	}

	// 直接从Dictionary里获取
	CFRunLoopRef loop = CFDictionaryGetValue(loopsDic, thread);

	if(!loop) {
		loop = _CFRunLoopCreate();
		CFDictionartSetValue(loopsDic, thread, loop);
		//注册一个回调，当线程销毁时，顺便也销毁对应的RunLoop
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

苹果是不允许直接创建RunLoop的，只能通过：
```objc
CFRunLoopRef CFRunLoopGetMain() {
	return _CFRunLoopGet(pthread_main_thread_np());
}

CFRunLoopRef CFRunLoopGetCurrent() {
	return _CFRunLoopGet(pthread_self());
}
```

这两个函数来获取当前或者某个线程的RunLoop。根据上面的源码可以得出结论：

	RunLoop创建是发生在第一次获取之前，若从未获取，则永远不会创建。RunLoop销毁是在线程结束之前，除主线程外，宿友RunLoop只能在当前线程中获取，主线程的RunLoop可以在任意线程中获取。

### Models in RunLoop

一个RunLoop包含若干个Modes，每个Modes又包含若干个 Source/Time/Observer。每次调用RunLoop的主函数时，只能指定其中一个Mode，这个Mode被称作CurrentMode。如果需要切换Mode，只能退出Mode，再重新指定一个Mode进入。这样做主要是为了分隔开不同组的Source/Timer/Observer, 让其互不影响。

1. Source:

	- Source是事件发生的地方。
	- Source分为Source0与Source1，Source0只包含一个函数指针作为回调，且不能主动唤醒RunLoop，使用时需要先动手将其标记为待处理，然后手动唤醒RunLoop；Source1也是包含函数指针，同时包含一个mach_por。它可以主动唤醒RunLoop，用于内核与其他线程发送消息。

2. Timer:

	- Timer是基于时间的出发起，它和NSTimer是toll-free bridged的，可以混用。其包含一个时间长度和一个回调（指针函数）。当其加入到RunLoop时，RunLoop会注册对应的时间点，当时间点到，RunLoop会被唤醒以执行那个回调。

3. Observer:

	- Observer 是观察者，用于观察RunLoop各种状态。它也包含函数指针，用于状态改编时回调通知外部。

	```objc
	typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
		kCFRunLoopEntry             = (1UL << 0), //即将进入Loop
		kCFRunLoopBeforeTimers      = (1UL << 1), //即将处理Timer
		kCFRunLoopBeforesOURCES     = (1ul << 2), //即将处理Source
		kCFRunLoopBeforeWaiting     = (1UL << 5), //即将进入休眠
		kCFRunLoopAfterWaiting      = (1UL << 6), //从休眠中唤醒
		kCFRunLoopExit              = (1UL << 7), //即将推出loop
	} 
	```

上面的Source/Timer/Observer被统称为mode item，一个item可以同时加入多个mode。单一个item被重复加入同一个mode时是不会有效果的。如果mode中一个item都没有，那么RunLoop会直接退出，不进入循环。

CFRunLoopMode 和 CFRunLoop 的结构大致如下：

``` objc
	struct __CFRunLoopMode {
		CFStringRed _name;
		CFMutableSetRef _source0;
		CFMutableSetRef _source1;
		CFMutableArrayRef _observers;
		CFMutableArrayRef _timers;
		...
	}

	struct __CFRunLoop {
		CFMutableSetRef _commonModes;
		CFMutableSetRef _commonModesItems;
		CFRunLoopModeRef _currentMode;
		CFMutableSetRef _modes;
		...
	}
```

这里有个概念叫`CommonModes`
一个Mode可以将自己标记为Common属性（通过将其ModeName 添加到 RunLoop的commonMoeds中）
每当RunLoop的内容发生变化是，RunLoop都会自动将_commonModesItems哩的Source/Observer/Timer 同步到具有Common标记的所有Mode里。

举例：主线程的RunLoop里有两个预置的Mode: kCFRunLoopDefaultMode 和 UITrackingRunLoopMode。 这两个Mode都已经被标记了Common属性。DefaultMode是App平时所处的状态，TrackingRunLoopMode是追踪ScrollView滑动的状态。当你床架哪一个Timer添加到DefaultMode时，Timer会得到重复的回调，但此时滑动一个TableView时，RunLoop的mode会切换为TrackingRunLoopMode，这时Timer就不会被调用，并且也不会影响到滑动操作。
有时你需要一个Timer，在两个Mode中都可以得到回调，一种办法是将Timer分别加入到两个Mode。还有一种方法，就是将Timer加入到顶层的RunLoop的commonModeItems中。commonModeItems被RunLoop自动更新到所有具有Common属性的Mode中。

### RunLoop 处理事件的逻辑

1. 通知Observer：即将进入Loop ————> Observer
2. 通知Observer：即将处理Timer ————> Observer
3. 通知Observer：将要处理Source0 ————> Observer
4. 处理Source0 ————> Source0
5. 如果有Source1，跳到第9 ————> Source1
6. 通知Observer：线程即将休眠 ————> Observer
7. 休眠，等待唤醒 ————> Source0(port)/Timer/外部手动唤醒
8. 通知Observer：线程被唤醒 ————> Observer 
9. 处理唤醒时收到的消息，之后跳会2 ————> Timer/Source1
10. 处理Ovserver：即将退出Loop ————> Observer
