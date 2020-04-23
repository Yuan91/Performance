iOS 项目优化
安装包压缩/启动优化/滑动卡顿/内存优化/性能检测/电量、网络优化/离屏渲染


一、安装包瘦身
1.图片压缩
2.多target工程，asset分开引用
3.清除无用的资源文件/代码
4.编译器设置：
Strip Linked Product、Make Strings Read-Only、Symbols Hidden by Default设置为YES
去掉异常支持，Enable C++ Exceptions、Enable Objective-C Exceptions设置为NO， Other C Flags添加-fno-exceptions

二、性能优化
1.性能优化新的要求
在iPhone5 时代，加载一个1000 条数据的tableview，其中cell的imageView和textLabel 都产生离屏渲染的情况下，
快速滑动列表，它的FPS在15左右。但是到了iPhone Xs Max的时代，同样的代码，FPS却能达到55，即便是Xs Max，GPU使用依然高达80%。
在当前设备性能普遍强大的情况下，列表流畅的滑动已经不是问题，如何在保证60FPS的同时，CPU和GPU能有一个较低的消耗才是我们要研究的。

三、关于离屏渲染
如何检测离屏渲染？
模拟器：选择 Debug-Color Off-Screen Renderd
真机：Debug-View Debug - Render 
如果界面呈现出黄色表示出现离屏渲染，这些图层很可能需要用shadowPath或者shouldRasterize来优化

当shouldRasterize设成true时，layer被渲染成一个bitmap，并缓存起来，等下次使用时不会再重新去渲染了。实现圆角/阴影本身就是在做颜色混合（blending），如果每次页面出来时都blending，消耗太大，这时shouldRasterize = yes，下次就只是简单的从渲染引擎的cache里读取那张bitmap，节约系统资源。这种做法就是牺牲内存，解放GPU。
而光栅化会导致离屏渲染，影响图像性能，那么光栅化是否有助于优化性能，就取决于光栅化创建的位图缓存是否被有效复用，而减少渲染的频度。当你使用光栅化时，你可以开启“Color Hits Green and Misses Red”来检查该场景下光栅化操作是否是一个好的选择。
如果光栅化的图层是绿色，就表示这些缓存被复用；如果是红色就表示缓存会被重复创建，这就表示该处存在性能问题了。
参考文章：https://www.cnblogs.com/feng9exe/p/10334751.html

对于工程的检测：圆角、阴影会触发离屏渲染。

其他类似还可以调试
Color Blended Layers 、
Color Copied Images

Color Misaligned Images
控件的坐标值转换为像素的时候，不是整数会被高亮显示。典型的例子是在2x的屏幕上，设置的坐标中有奇数值。导致显示视图的时候需要对没对齐的边缘进行额外混合计算，影响性能。强大的A系列芯片，不至于连这点额外的计算都不能处理，所以这个注意就好
参考文章：https://www.jianshu.com/p/38cf9c170141

四、启动优化研究
