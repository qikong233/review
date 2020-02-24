### 图片裁剪

	只有UIImage的情况

#### 创建新的ImageContext

	```
	UIGraphicsBeginImageContext(targetSize);
	CGRect targetRect;
	[originImage drawInRect:targetRect];
	UIImage *targetImage = UIGraphicsGetImageFromCurrentImageContext();
	UIGraphicsEndImageContext();
	```

#### 通过CGImage进行裁剪

	```
	UIImage *targetImg = [UIImage imageWithCGImage:CGImageCreateWithImageInRect([img CGImage], targetRect)];
	```
	
