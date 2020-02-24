### iOS Player

#### MPMoviePlayerViewController/MPMoviePlayerController

	在iOS之前，iOS播放视频文件一般使用MPMoviePlayerViewController和MPMoviePlayerController。

	前者继承自UIViewController，后者继承自NSObject。MPMoviePlayerViewController里面包含一个MPMoivePlayerController。

	想要使用这两者，要引入MediaPlayer.framework

	MPMoviePlayerViewController中只有三个方法和一个属性：

	```objc
		- (instancetype)initWithContentURL:(NSURL *)contentURL

		- (void)presentMoviePlayerViewControllerAnimated:(MPMoviePlayerViewController *)moviePlayerViewController

		- (void)dismissMoviePlayerViewControllerAnimated

		@property(nonatomic, strong) MPMoviePlayerController *moviePlayer;
	```

	从根本意义上来说，MPMoviePlayerViewController的方法实现就是MPMoviePlayerController中的对应初始化，加入视图的方法，前者只是对后者进行一个简单的封装。

	如果要对播放视频的属性进行操作，可以通过设置MPMoviePlayerViewController.moviePlayer来实现。MPMoviePlayerViewController中可以的修改的参数有很多。

	以下是关于播放视频的监听事件，注册之后，当对应状态改编时就可以收到对应的通知。
	```objc
	// 当视频缩放比例改变时候
	NSString * const MPMoviePlayerScalingModeDidChangeNotification

	// 当视频播放结束时
	NSString * const MPMoviePlayerPlaybackDidFinishNotification

	// 当用户退出视频时
	NSString * const MPMoviePlayerPlabackDidFinishReasonUserInfoKey

	// 当回调状态改变时
	NSString * const MPMoviePlayerPlaybackStateDidChangeNotification

	// 当网络加载状态改变时候
	NSString * const MPMoviePlayerLoadStateDidChangeNotification

	//当前播放视频改变时

	NSString * const MPMoviePlayerNowPlayingMovieDidChangeNotification 

	// 当进入全屏或者退出全屏

	 NSString * const MPMoviePlayerWillEnterFullscreenNotification 

	 NSString * const MPMoviePlayerDidEnterFullscreenNotification 

	 NSString * const MPMoviePlayerWillExitFullscreenNotification 

	 NSString * const MPMoviePlayerDidExitFullscreenNotification 

	 NSString * const MPMoviePlayerFullscreenAnimationDurationUserInfoKey 

	 NSString * const MPMoviePlayerFullscreenAnimationCurveUserInfoKey 

	// 在appleTv或者音响上播放状态改变时

	 NSString * const MPMoviePlayerIsAirPlayVideoActiveDidChangeNotification 

	// 当准备状态改变时

	 NSString * const MPMoviePlayerReadyForDisplayDidChangeNotification 
	```

	播放本地路径下的视频的实例代码如下

	```objc
	- (void)Play:(NSString *)resName TypeName:(NSString *)type {
		NSString *path = [[NSBundle mainBundle] pathForResource:resName ofType:type];
		if (nil == path) {
			return;
		}

		NSURL *url = [NSURL fileURLWithPath: path];

		MPMoviePlayerViewController *_moviePlayer = [[MPMoviePlayerViewController alloc] initWithContentURL:url];
		_moviePlayer.moviePlayer.controlStyle = MPMovieControlStyleNone;

		[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(movieFinishedCallback:) name:MPMoviePlayerPlaybackStateDidChangeNotification object: [_moviePlayer moviePlayer]];

		[_moviePlayer.moviePlayer play];

		[[self GetRootViewController] presentMoviePlayerViewControllerAnimated:_moviePlayer];

		_moviePlayer = nil;
	}

	- (void)movieFinishedCallback:(NSNotification *)notifi {
		MPMoviePlayerController *player = [notifi object];
		[[NSNotificationCenter defaultCenter] removeObserver:self name:MPMoviePlayerPlaybackStateDidChangeNotification object:player];
		[player stop];
		[[self GetRootViewController] dismissMoviePlayerViewControllerAnimated];
	}
	```

#### AVPlayerController

	在iOS9之后，上述的MPMoviePlayerController就被苹果弃用了
	使用AVPlayerViewController来替代，简而言之就是MPMoviePlayerController使用更简单，功能不如AVPlayer强大，而AVPlayer使用稍微麻烦点，不过功能更加强大。

	```objc
	NSString *filePath = [[NSBundle mainBundle] pathForResource:@"backspace" ofType:@"mov"];
	NSURL *sourceMovieURL = [NSURL fileURLWithPath:filePath];

	AVAsset *moveAsset = [AVURLAsset URLAssetWithURL:sourceMovieURL options: nil];

	AVPlayItem *playerItem = [AVPlayerItem playerItemWithAsset:movieAsset];

	AVPlayer *player = [AVPlayer playerWithPlayerItem:playerItem];

	AVPlayerLayer *playerLayer = [AVPlayerLayer playerLayerWithPlayer:player];

	playerLayer.frame = self.view.layer.bounds;

	playerLayer.vieoGravity = AVLayerVideoGravityResizeAspect;

	[self.view.layer addSublayer:playerLayer];

	[player play];
	```

	上方的代码实现的效果其实和MPMoviePlayerController实现的是一样的，AVPlayer更强大的地方是它有对应的方法去调节视频的音量以及视频的进度

#### AVAsset

	AVAsset 用于建模同步视听媒体：如视频和声音的抽象类。

	AVAsset定义了组成资源的轨道的集合属性。使用指向媒体资源（本地或远程）URL初始化AVAsset

	AVAsset是一个抽象类，因此创建实例中所示的AVAsset实际上是创建一个名为AVURLAsset的具体子类的实例。在许多情况下，这是创建AVAsset的合适方法，但是当需要对其初始化进行更粒度的控制时，也可以直接实例化NSURLAsset。
	AVURLAsset的初始化程序接受一个选项字典，使该字典可以根据特定用例定制资源的初始化。例如，如果正在为HLS流创建AVAsset，则可能希望在用户连接到蜂窝网络时阻止其检索其媒体。
	```objc
	NSURL *url = [NSURL URLWithString:@""];
	NSDictionary *options = @{AVURLAssetAllowCellularAccessKey: @NO};
	AVURLAsset *urlAsset = [AVURLAsset URLAssetWithURL: url options: options];
	```
