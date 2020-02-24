### SDWebImage

#### 调用顺序

1. 通用方法 - (void)sd_setImageWithURL:(nullable NSURL *)url placeholderImage(nullable UIImage *)placeholder

2. SDWebImageManager 单例调用
```objc
- (nullable id<SDWebImageOperation>)loadImageWithURL:(nullable NSURL*)url options:(SDWebImageOptions)options progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock completed:(nullable SDInternalCompletionBlock)completedBlock;

// Manager两个单例
@property(strong, nonatomic, readonly, nullable) SDImageCache *imageCache; // 处理缓存
@property(strong, nonatomic, readonly, nullable) SDWebImageDownloader *imageDownloader; // 处理网络下载
```
- Manager上面那个方法下面初始化
	```objc
	__block SDWebImageCombinedOperation *operation = [SDWebImageCombinedOperation new];
	```
	
- SDImageCache 缓存单例
	```objc
	operation.cacheOperation = [self.imageCache queryCacheOperationForKey ...]
	// 二级缓存查找方法，把队列返回复制给Operation的Cache字段
	```
	- 内存缓存NSCache 磁环缓存FileManager文件存储 无论找到与否都返回， 只是image是否为空的区别

- SDWebImageDownloader 下载单例
	
	在上面queryCache的到左之后根据查找的情况调用[self.imageDownloader downloadImageWithURL...]

	调用addProgressCallback该方法下的Block是创建SDWebImageDownloaderOperation 下载队列用的
