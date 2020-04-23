# iOS UIView 绘制流程分析

UIView 绘制流程如下:

![avatar](https://s1.ax1x.com/2020/04/18/JegDrn.png)

1.当我们调用`[UIView setNeedsDislpay]`的时候,它会调用`CALayer`的同名方法`setNeedsDislpay`.这个时候视图并没有开始绘制,而是给视图添加一个脏标记放到全局的容器中,等到当前`runloop`处理完事件处于`beforeWating`或者`exit`的状态,才开始绘制.

**思考一下为什么,会是这样的?**

因为当我们修改`UIView`的`frame/bounds/color`等属性的时候,其实是调用`CALayer`的同名属性,而`CALayer`内部是并没有这些属性的实现,它会在运行时调用`resolveInstanceMethod`动态添加方法,并把这些属性值保存在一个字典中.所以这些属性的修改是相对耗时的操作.

把他们放到`runloop`中等到它处理完唤醒事件,处于即将休眠(`beforeWaiting`)或者将要退出循环(`exit`)的状态一次性的来处理UI更新,既可以避免每次属性变更都要执行更新的性能浪费,又能保证属性设置的先后顺序不会影响视图的绘制.

**那么我们再来思考一下为什么要用`UIView`来包装一下`CALayer`,而不是直接使用`CALayer`来添加视图呢?**

在iOS开发中,Apple推荐我们使用MVC框架来开发,其实它自己的系统框架中也处处透露着MVC的身影,比如我们的UIKit.
`UIView`是`CALayer`的代理,负责处理用户输入事件.在这里`CALayer`相当于MVC中的View,而`UIView`相当于`Controller`的角色.

另外一个重要的原因就是这样设计方便跨平台,iOS设备和Mac设备上,对于用户输入事件的检测是不同的,它们分别对应视图控件类是`UIView`和`NSView`,但是他们都是依赖`CALayer`进行显示的.这样职责分离之后,就方便的Apple的工程师进行代码代码复用,他们只需要编写不同平台的事件处理类,而后续图层的绘制(display)/准备(perpare)/图层树提交(commit)等流程的代码是可以复用的.



2.绘制方式判断

当`runloop`处于`beforeWaiting`或者`exit`状态时,它会调用`[CALayer displayLayer]`方法,该方法会判断layer的delegate是否响应了`displayLayer`方法,如果响应了方法就是执行自定义绘制流程,否则就会进入系统绘制流程.

当我们使用`UIView`创建视图的时候,它是作为`CALayer`代理的存在,所以只要我们在`UIView`中实现`displayLayer`方法就会进入自定义绘制的流程.在一些性能要求高的场景中,我们可以在该方法中执行异步绘制,将绘制操作从主线程中分离开来.

自定义绘制的关键在于创建一个bitmap将其赋值给`layer.contents`.
示例:绘制一个图片和一段文字到的UIView
```
AsyncDrawView *view = [[AsyncDrawView alloc]init];
self.asyncView = view;
view.bounds = CGRectMake(0, 0, 150, 150);
view.center = self.view.center;
view.backgroundColor = [UIColor redColor];
[self.view addSubview:view];
```

```
//AsyncDrawView.m
- (void)displayLayer:(CALayer *)layer{
    if ([layer isEqual:self.layer] == NO) {
        return;
    }
    CGRect bounds = self.bounds;
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        layer.contentsScale = [UIScreen mainScreen].scale;
        UIGraphicsBeginImageContextWithOptions(layer.bounds.size, layer.isOpaque, layer.contentsScale);
        CGContextRef context = UIGraphicsGetCurrentContext();
        CGContextSaveGState(context);
        CGContextSetFillColorWithColor(context, [UIColor purpleColor].CGColor);
        CGContextFillRect(context, bounds);
        //绘制 图片到 View
        UIImage *aImage = [UIImage imageNamed:@"Anchor"];
        CGRect imgRect = CGRectMake(0, 0, 40, 40);
        [aImage drawInRect:imgRect];
        //绘制 文字到 View
        NSString *text = @"通过异步绘制将图片和文字绘制到UIView上";
        NSDictionary *attr = @{NSFontAttributeName:[UIFont systemFontOfSize:15],
                               NSForegroundColorAttributeName:[UIColor whiteColor]
        };
        //超出layer.bounds的内容会被裁剪掉
        CGFloat textX = CGRectGetMaxX(imgRect) + 5;
        CGRect textRect = CGRectMake(textX, 0,bounds.size.width - (textX), 100);
        [text drawInRect:textRect withAttributes:attr];
        CGContextRestoreGState(context);
        //生成图片
        CGImageRef cgimage = CGBitmapContextCreateImage(context);
        UIImage *image = [UIImage imageWithCGImage:cgimage];
        dispatch_async(dispatch_get_main_queue(), ^{
            layer.contents = (__bridge id _Nullable)(image.CGImage);
        });
    });
}
```

自定义绘制效果
![avatar](https://s1.ax1x.com/2020/04/18/JeIOKS.png)

# 系统绘制流程分析

系统绘制流程图

![avatar](https://upload-images.jianshu.io/upload_images/1311960-ca606fee8fd18646.png?imageMogr2/auto-orient/strip|imageView2/2/w/481)

1.进入到系统绘制流程,`CALayer`开始实际的绘制,此时它会创建一个`backing store`,他是用来绘制到屏幕上的位图.

执行自定义绘制的时候，如`CALayer`子类的`-drawInContext`或者它的代理`-drawLayer:inContext:`中绘制的位图，都保存在这个`backing store`中

2.`layer`会判断是否有代理,如果我们使用的是`UIView`,那么`UIView`对象就是`layer`的代理

3.如果`layer`有代理,会调用它实现的`drawLayer:inContext`方法,进而调用`UIView`的`drawRect`方法.

关于`drawLayer:inContext`

①如果`layer`实现了我们前面说的`displayLayer`方法,那么这个方法是不会被调用的.

②`drawLayer:inContext`是系统绘制的默认实现,所以当实现了这个方法的时候记得需要根据实际情况判断是否需要调用`[super drawLayer:layer inContext:ctx]`

③该方法执行完之后,会执行`drawRect`方法

关于`drawRect`方法

①这个方法默认是一个空实现,即什么也不做.它只是提供了一个入口,给与开发者在系统绘制之后,使用`UIKit`以及`Core Graphic`做一些自定义绘制

②当这个方法调用的时候,系统已经为开发者配置好了绘制环境和上下文.

③`drawRect`方法的触发时机:当视图第一次显示到屏幕上,或者视图显示部分不可用的视图(如`setNeedsDisplay`,`setNeedsDisplayInRect`的调用)

4.如果`layer`没有代理,则会执行`CALayer`的`drawInContext`开始系统绘制

5.至此`layer`的绘制完成,生成`backing store`也就是位图.此后会通过`Core Animation`将位图提交到`GPU`进行图层合成和纹理渲染.