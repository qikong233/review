### GPUImage

#### addTarget 方法逻辑

	source(图片/视频) ——target——> filter ——target——> show(GPUImageView/GPUImageMovieWriter)

	如果做滤镜的叠合，其实就是多个Filter串联在一起

	source(图片/视频) ——target——> filter ——target——> otherfilter ——target——> show(GPUImageView/GPUImageMovieWriter)

	如果这些滤镜中，要做混合效果（强光混合之类，就是需要2个输入源的话）：

	source(图片/视频) ——target——> filter ——target——> blendFilter ——target——> show(GPUImageView/GPUImageMovieWriter)

								|
				      GPUImagePicture ——target——> 
