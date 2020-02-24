### UIImagePickerController

#### Usage

1. 创建
2. 设置允许编辑
	ImagePickerController.allowEditing = YES;
3. 设置来源类型（相册/相机
	ImagePickerControlelr.sourceType = UIImagePickerControllerSourceTypeCamera/PhotoLibrary

	- 相机设置

		- 设置视频拍摄时长，默认10s `videoMaximunDuration`
		- 设置相机类型 `ImagePickerController.meidaTypes = @[(NSString *)kUTTypeMovie, @(NSString *)kUTTypeImage]`
		- 设置视频上传质量 `ImagePickerController.videoQuality = UIImagePickerControllerVideoQualityTypeHigh`
		- 设置摄像头模式 `ImagePickerController.cameraCaptureMode = UIImagePickerControllerCameraCaptureMovieVideo`

	- 相册设置

		- 设置相册媒体类型 `ImagePickerController.meidaTypes = [NSArray arrayWithObjects: @"public.movie", nil]` 
4. 设置代理回调
	
	- 代理 `imagePickerController:didFinishPickingMediaWithInfo:` 
	- 获取照片或者视频

	```
	NSString *mediaType = [info objectForKey:UIImagePickerControllerMediaType];
       		// 选取的是图片
    	} else {
        	// 选取的是视频
        	NSURL *url = info[UIImagePickerControllerMediaURL];
    	}
    	```

