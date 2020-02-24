### AVKit

#### 视频播放

AVKit是iOS8.0开始使用的视频播放API

AVKit还是以AVFoundation的形式来使用

	MediaPlayer是基于AVFoundation的基础上进行封装了
	提供了带视图和不带视图的两种方式
	现实开发中，系统默认封装好的MPMoviePlayerViewController(带有视图)很少被使用

#### 知识点

- Using Asset
	理解为：数据来源
	```objc
	NSURL *url = ...(url)
	AVURLAsset *asset = [[AVURLSet alloc] initWithURL:url options:nil];
	```

- 获得一个视频的图像
	使用AVSetImageGenerator类来实现
	用来生成图像序列

- Playback
	我们在播放视频时可以使用AVPlayer和AVQueuePlayer播放,AVPlayer是AVQueuePlayer的父类
	- 先创建一个路径
	- 可以使用AVPlayerItem加载路径
	- 使用AVPlayer播放文件

	当让我们还可以控制它的播放速度
	使用rate属性，它是一个介于0.0 —— 1.0之间的数

	我们也可以播放多个项目：
	```objc
	NSArray *items = //设置一个播放的组合
	AVQueuePlayer *queuePlayer = [[AVQueuePlayer alloc] initWithItems: items]
	```

- Media capture
	我们可以配置预设图片的质量和分辨率

	AVCaptureSessionPresetHigh High最高的录制质量，每台设备不同
	AVCaptureSessionPresetMedium Medium基于无线分享的，实际值可能会改编
	AVCaptureSessionPresetLow Low基于3g分享的
	AVCaptureSessionPreset640x480 640x480 VGA
	AVCaptureSessionPreset1280x720 1280x720 720p HD
	AVCaptureSessionPresetPhoto Photo完整的照片分辨率，不支持视频输出

#### 添加水印

使用AVFoundation为视频添加水印步骤如下：

1.拿到视频和音频资源

2.创建AVMutableComposition对象

3.往AVMutableComposition对象添加视频资源，同时设置视频资源的时间段和插入点

4.往AVMutableComposition对象添加音频资源，同时设置音频资源的时间段和插入点

5.往AVMutableCoposition对象添加要追加的音频资源，同时设置音频资源的时间段，插入点和混合模式

6.导出视频。导出视频依然使用AVAssetExportSession

	必须判断是否有音频[[videoAsset tracksWithMediaType:AVMediaTypeAudio] count] > 0，否则会出现问题。
	框架定义了一个名为AVURLAssetPreferPreciseDurationAndTimeingKey的选项，选项带有@YES值可以确保党资源的属性使用AVAsynchronousKeyValueLoading协议载入时可以计算出准确的时长和时间信息。虽然使用这个option时还会载入过程增添一些额外的开销，不过这可以保证资源正处于合适的编辑状态。

#### 视频合成-插入图片了动画

1.获取视频资源AVURLAsset

2.创建自定义合成对象AVMutableComposition，定义为可变组件。

3.在可变年组建中添加资源数据，也就是轨道AVMutalbeCompositionTrack（一般添加2种：音频轨道和视频轨道

4.创建视频组件AVMutableVideoComposition，这个类是处理视频中要编辑的东西。可以设定为所需视频的大小、规模以及帧的持续时间。以及管理并设置视频组件的指令。

5.创建视频组件的指令AVMutableVideoCompositionInstruction，这个类主要用于管理应用层的指令。

6.创建视频应用层的指令AVMutableVideoCompositionLayerInstruction 用户管理视频框架应该如何被应用和组合，也就是说是子视频在总视频中出现和消失的时间、大小、动画等。

7.创建视频导出绘画对象AVAssetExportSession，主要是根据videoComposition去创建一个新的视频，并输出到一个指定的文件路径中去。        

补充：由于4、5、6步骤直接的关联性，要修改下创建顺序6->5->4这样去创建代码更加清晰。
