### 常见用法

#### Autorelease Pool

App启动后，苹果在主线程Runloop里注册了两个Observer，其回调都是_wrapRunLoopWithAutoreleasePoolHandler()

第一个 Observer 监听的事件是 Entry(即将进入Loop)，其回调会调用_objc_autoreleasePoolPush() 创建自动释放池。其order是-2147483647，优先级最高，保证创建释放吃发生在其他所有回调之前。

第二个 Observer 监听两个时间：BoforeWaiting 时调用_objc_autoreleasePoolPop() 和_objc_autoreleasePoolPosh()释放就的池并创建新的池；Exit时调用 _objc_autoreleasePoolPop()来释放自动释放池。这个Observer的order是2147483647，优先级最低，保证其释放池子的发生在所有回调之后。

在主线程执行的代码，通常是卸载诸如时间回调、Timer回调内的。这些回调会被Runloop创建好的AutoreleasePool环绕着，所以不会出现内存泄漏，开发者也不必显示创建Pool。

#### 事件响应

苹果注册了一个Source1（基于mach port的）用来接收系统事件，其回调函数为

__IOHIDEventSystemClientQueueCallBack()

当一个硬件时间（触摸/锁屏/摇晃等）发生后，首先由IOKit.framework生成一个IOHIDEvent事件并由SpringBoard接收。
SpringBoard只接收按键（锁屏/静音），触摸，加速，接近传感器几种Event，随后调用mach port转发给需要的App进程。
随后苹果注册的Source1就会触发回调，并调用_UIApplicationHandleEventQueue()进行应用内部的分发。

_UIApplicationHandleEventQueue()会把IOHIDEvent处理并包装成UIEvent进行处理或分发，其中包括UIGesture处理屏幕旋转，发送给UIWindows等。通常时间比如UIButton点击、touchesBegin/Move/End/Cancel 时间都是在这个回调中完成的。

#### 界面更新

当在操作UI时，比如改变了Frame、更新了UIView/CALayer的层次时，或者手动调用了UIView/CALayer的setNeedsLayout/setNeedsDisplay方法后，这个UIView/CALayer就被标记为待处理，并被提交到一个全局的容器去。

苹果注册了一个Observer监听BeforeWaiting和Exit事件，回调去执行一个很长的函数：_ZN2CA11Transaction17ovserver_callbackEP19_CFRunloopObservermPv()。这个函数会遍历所有待处理的UIView/CAlayer以执行实际的绘制和调整，并更新UI界面。

#### 定时器

NSTiemr 其实就是CFRunLoopTimerRef， 他们之间是toll-free bridged的。 一个NSTimer注册到RunLoop后，RunLoop会为其重复的时间点注册号事件。例如10.00 10.10 10.20这几个时间点。RunLoop为了节省资源，并不会在非常准确的时间点回调这个Timer。Timer有个属性叫做Tolerance（宽容度），标示了当前时间点到后，容许有多少最大误差。

如果某个时间点被错过了，例如执行了一个很长的任务，则那个时间点的回调也会跳过去，不会延后执行。剧比如等公交，如果10.10时玩手机错过了那个点的公交，那么只能等10.20的一趟了。

CASdisplaylink是一个和屏幕刷星率一致的定时器（但实际实现原理更复杂，和NSTimer并不一样，其内部实际是操作了一个Source）。如果在两次屏幕刷新之间执行了一个长任务，那其中就会有一帧被跳过去（和NSTimer相似），造成界面卡顿的感觉。在快速滑动tableview时，即使一帧卡顿也会让用户有所察觉。

#### PerformSelector

当调用NSObject 的 performSelecter:afterDelay:后，实际上其内部会创建一个Timer并添加到当前线程的Runloop中。所以如果当前线程没有RunLoop，则这个方法会失效。

当调用performSelector:onThread:时，实际上会创建一个Timer加到对应的线程去，同样的，如果没有对应线程的RunLoop 该方法也会失效。

#### GCD

GCD 提供的某些接口也用到了RunLoop，例如dispatch_async()

当调用 dispatch_async(disptch_get_main_queue(), block)时，libDispatch会向主线程的RunLoop发送消息，RunLoop会被唤醒，并消息中获取到这个block，并在回调__CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__()里执行这个block。但这个逻辑仅限于dispatch到主线程，dispatch到其他线程仍然是libDispatch处理的。

#### AFN

AFURLConnectionOperation 这个类是基于NSURLConnection构建的，其希望能够在后台线程接受Delegate回调。为此AFNetworking单独创建一个线程，并在这个线程中启动一个RunLoop：

```objc
+ (void)networkRequestThreadEntryPoint:(id)__unused object {
	@autoreleasepool{
		[[NSThread current] setName:@"AFNetworking"];
		NSRumLoop *runLoop = [NSRunLoop currentRunLoop];
		[runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
		[runLoop run];
	}
}

+ (NSThread *)networkRequestThread {
	static NSThread *_networkRequestThread = nil;
	static dispatch_once_t oncePredicate;
	dispatch_once(&oncePredicate, ^{
		_networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
		[_networkRequestThread start];
	})
	return _networkRequestThread;
}
```

#### AsyncDisplayKit

AsyncDisplayKit 是 Facebook 推出的用于保持界面流畅性的框架，其原理大致如下：

UI 线程中一单出现繁重的任务就会导致界面卡顿，这类任务通常分类为：排版、绘制、UI对象操作。

1. 排版通常包括计算视图大小、计算文本高度、重新计算子视图的排版等操作。
2. 绘制一般有文本的绘制（例如CoreText）、图片绘制（例如预先解压）、原色绘制（Quartz）等操作。
3. UI对象操作通常包括UIView/CAlayer 等UI对象的创建、设置属性和销毁。

其中前两类操作可以通过各种方法扔到后台线程执行，而最后一类操作只能在主线程完成，并且有时后面的操作需要依赖前面的操作结果（例如TextView创建时可能需要提前计算文本的大小）。ASDK所做的，就是尽量将能放到后台的任务放入后台，不能的则尽量推迟（例如视图的创建、属性的调整）

为此，ASDK创建了一个名为ASDisplayNode的对象，并在内部封装了UIView/CALayer，它具有和UIView/CALayer相似的属性，例如frame、backgroundColor等。所有这些属性都可以在后台线程更改，开发者可以只通过Node来操作其背部的UIView/CALayer，这样就可以排版和绘制放入后台。但是无论怎么操作，这些属性总需要在某个时刻同步到主线程的UIView/CALayer去。

ASDK 仿造 QuartCore/UIKit 框架的模式，实现了一套类似的界面更新的机制：即在主线程的RunLoop中添加一个Observer，监听了kCFRunLoopBeforeWaiting和kCFRunLoopExit时间，在收到回调时，遍历所有之前放入队列的待处理的任务，然后一一执行。
