# GeometryReader

> 一个可以根据其自身大小和坐标空间定义内容的容器视图

可以在GeometryReader中获取到子View的位置信息等等。但是在使用的时候需要给GeomertyReader一个宽高

## 使用

```swfit
GeometryReader {geometry in 
	View()
	  .rotate3DEffect(Angle(Double(geometry.frame(in: .global).minX, axis: (x: 0, y: 10, z: 0))))
	  .frame(width: 100, height: 100)
}
.frame(width: 100, height: 100)

```
