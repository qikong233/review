### iOS渲染与绘制

#### 理解iOS渲染框架的运作机制

UIKit是常用的框架，显示、动画都通过CoreAnimation。CoreAnimation是核心动画，依赖于OpenGL ES做GPU渲染，CoreGraphics做CPU渲染；最低成的GraphicsHardWare是图形硬件。

#### 理解CALayer
在iOS中，所有的视图都从一个叫UIView的基类派生而来，UIView可以处理触摸类事件，可以支持基于Core Graphics绘图，可以做仿射变换（例如旋转或者缩放），或者简单的类似于滑动或者渐变的动画。

CALayer类在概念上和UIView类似，同样也是一些层级关系树管理的矩形块，同样也可以包含一些内容（像图片、文本或者背景色），管理子图层的位置。它们有一些方法和属性用来做动画和变换。和UIView最大的不同是CALayer不处理用户的交互。CALayer并不清楚具体的响应链。

UIView和CALayer是一个平行的层级关系，每一个UIView都有一个CALyaer实例的图层属性，也就是所谓的backing layer，视图的职责就是创建并管理这个图层，以确保子视图在层级关系中添加或者被移除的时候，他们关联的图层也同样对应在层级关系树当中有相同的操作。实际上这些背后关联的Layer图层才是真正用在屏幕上显示和做动画，UIView仅仅是对它的一个封装，提供了一些iOS类似于处理触摸的具体功能，以及CoreAnimation底层方法的高级接口。

UIView的Layer在系统内部，被维护着三分同样的树形数据结构，分别是：

- 图层树（这里是代码可以操纵的，设置属性的最终值都会立刻在这里更新）

- 呈现树（是一个中间层，系统就在这一层上更改属性，进行各种渲染操作。比如一个动画的更改是alpha值从0到1，那么在逻辑树上此属性会被立刻更新为最终值1，而在动画树上会根据设置的动画时间从0足部变化到1）

- 渲染树（其属性值就是当前正本显示在屏幕上的属性值）

#### 渲染驱动器：CADisplayLink

Runloop中定时器

NSTimer其实就是CFRunLoopTimerRef。一个NSTimer注册到RunLoop后，RunLoop会为其重复的时间点注册好事件。

RunLoop为了节省资源，并不会在非常准别的时间点回调这个Timer，Timer有个属性叫做Tolerance（宽容度），标示了当前时间点到后，容许有多少最大误差。如果某个时间点被错过了，则那个时间点的回调也会跳过，不会延后执行。

RunLoop使用GCD的dispatch_source_t实现的Timer。当调用NSObject的performSelecter:afterDelay:后，实际上其内部会创建一个Timer并添加到当前线程的Runloop中。所以如果当前线程没有Runloop，则这个方法会失效。当调用performSelecter:onThread:时，实际上会创建一个Timer加到对应的线程去，同样的，如果对应的线程没有Runloop该方法也会失效。

CADisplayLink是一个和屏幕刷新率一致的定时器。如果在两次屏幕刷新之间执行了一个长任务，那其中就会有一帧被跳过去，造成界面卡顿的感觉。

#### 渲染流程

通常来过，计算机系统中CPU/GPU/显示器是以上这种方式协同工作的。CPU计算好内容提交到GPU，GPU渲染完成后将其渲染结果放入到缓冲区，随后视屏控制器会按照VSync信号入下图所示，逐行读取帧缓冲区的数据，经过可能的数据转换传递给显示器显示。

在VSync信号到来后，系统图形服务会通过CADisplayLink等机制通知App，App主线程开始在CPU中计算显示内容，比如视图的创建，布局计算，图片解码，文本绘制等。随后CPU会讲计算好的内容提交到GPU去，由GPU进行变换、合成、渲染。随后GPU回吧渲染结果提交到帧缓冲区去，等待下一次Vsync信号到来是显示到屏幕上。由于垂直同步的机制，如果在一个VSync时间内，CPU或者GPU没有完成内容提交，则那一秒就会被丢弃，等待下一次机会再显示，而这时屏幕会保留之前的内容不变。这就是界面卡顿的原因。从上图可以看到，CPU和GPU不论那个阻碍了显示路程，都会造成掉帧现象。所以开发时，也要分别对CPU和GPU压力进行评估和优化。

iOS的显示系统是由VSync信号驱动的，VSync信号由软件时钟生成，每秒发出60次（这个值取决设备硬件，比如iPhone真机上通常是59.97）。iOS图形服务接收到VSync信号后，会通过IPC通知到App内。App的Runloop在启动后悔注册对应的CFRunLoopSource通过mach_port接受传过来的始终信号通知，随后Source的回调驱整个App的动画与显示。

CoreAnimation在RunLoop中注册一个Observer，监听了Beforewaiting和Exit事件。当一个触摸时间到来时，RunLoop被唤醒，App中的代码会执行一些操作，比如创建和调整视图层级、设置UIView的frame、修改CALayer的透明度、为视图添加一个动画；这些操作最终都被CALayer标记，并通过CATransaction提交到一个中间状态去。当上面所有操作结束后，RunLoop即将进入休眠状态（退出）时，关注该事件的Observer都会得到通知。这时CoreAnimation注册的那个Observer就会在回调中，把所有的中间状态合并提交到GPU去显示；如果此处有动画，通过DispalyLink稳定的刷新机制会不断唤醒runloop，使得不断的有机会出发observer回调，从而根据时间来不断更新这个动画的属性值并绘制出来。

为了不阻塞主线程，CoreAnimation的核心是OpenGL  ES的一个抽象物，所以大部分的渲染是直接提交给GPU来处理。而Core Graphics、Quartz 2D的大部分绘制操作都是在主线程和CPU上同步完成的，比如自定义UIView的drawRect里用CGContext来画图。


