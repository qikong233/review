# Image

SwiftUI中的图片容器

- 默认大小为图片大小，设置frame需要先设置resizable

`Image("").resizable()`

- 系统图标库

`Image(systemName: "")`

系统图标图设置大小可以直接调用font

`Image(systemName: "").font(.system(size: 30, weight: .bold, ..))`

