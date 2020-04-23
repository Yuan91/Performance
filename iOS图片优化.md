# iOS 图片渲染优化
本文讲解异步解码和下采样的使用,以及与`SDWebImage`配合使用

* 异步解码
生成`UIImage`对象之后,图片并不会并不会立马解码,而是等到它设置到`UIImageView`后才会开始解码,这个解码默认在主线程执行,且会消耗大量的CPU,所以可以采用异步解码的方式,避免主线耗时操作

* 下采样
当图片大小明显大于`UIImageView`控件大小的时候,可以采用下采样生成一张缩略图降低内存使用.图片在内存中占用的大小,跟它的文件大小没有关系而是取决它的像素,一般的计算公式是**图片的宽 * 图片的高 * 4**(假设每个像素占用4个字节).
以一张266KB大小,1718 像素宽和 2048 像素高的图片为例,它占用内存计算公式:`1718 * 2048 * 4 / 1000000 = 14.07 MB`

## 图片渲染的流程
![avatar](https://github.com/rob2468/rob2468.github.io/raw/master/resources/figures/2018-08-22-image-rendering-pipeline.png)

加载一张图片到屏幕上,会有以下四个步骤

**1.加载**

这一步对应我们通过`imageNamed`或`imageWithContentsOfFile` 生成的`UIImage`,这个时候图片是未解码的压缩数据,这个数据被存储在`Data Buffer`中.
png/jpg 都是压缩后的数据,使用这种格式能够节省内存提高传输速度.

**2.解码**

当图片被设置到`UIImageView`中,如`imageView.image = image`,才会触发解码操作,这一步默认是在主线程执行的,而且是会消耗大量的CPU资源.所谓解码是指将图片的原始数据转化为可以显示在屏幕上的像素数据,也就是我们常说的位图.解码后的图片存储在`Image Buffeer`中,`Image Buffer`所占用的内存大小和图片的尺寸大小成正比,满足我们上面提到的公式.

**3.提交**

通过`CoreAnimation`将`Image Buffer`中的数据提交到`Frame Buffer`中,这里面存储的也是像素数据

**4.渲染**

时钟硬件会按照1s 60帧的速度从`Frame Buffer`中读取像素数据逐个的点亮屏幕上的像素点.


## 优化CPU使用:异步解码

目前大部分的网络图片加载库都实现了异步解码的操作,如`SDWebImage`,`YYKit`.
以`SDWebImage`为例,它在网络图片下载完之后会调用`decodedImageWithImage`方法,在子线程进行图片解码,其核心是调用`CGBitmapContextCreate` 生成一个context,从而用它来生成一个CGImageRef.详细代码如下:
```
+ (nullable UIImage *)decodedImageWithImage:(nullable UIImage *)image {
    if (![UIImage shouldDecodeImage:image]) {
        return image;
    }
    
    // autorelease the bitmap context and all vars to help system to free memory when there are memory warning.
    // on iOS7, do not forget to call [[SDImageCache sharedImageCache] clearMemory];
    @autoreleasepool{
        
        CGImageRef imageRef = image.CGImage;
        CGColorSpaceRef colorspaceRef = [UIImage colorSpaceForImageRef:imageRef];
        
        size_t width = CGImageGetWidth(imageRef);
        size_t height = CGImageGetHeight(imageRef);
        size_t bytesPerRow = kBytesPerPixel * width;

        // kCGImageAlphaNone is not supported in CGBitmapContextCreate.
        // Since the original image here has no alpha info, use kCGImageAlphaNoneSkipLast
        // to create bitmap graphics contexts without alpha info.
        CGContextRef context = CGBitmapContextCreate(NULL,
                                                     width,
                                                     height,
                                                     kBitsPerComponent,
                                                     bytesPerRow,
                                                     colorspaceRef,
                                                     kCGBitmapByteOrderDefault|kCGImageAlphaNoneSkipLast);
        if (context == NULL) {
            return image;
        }
        
        // Draw the image into the context and retrieve the new bitmap image without alpha
        CGContextDrawImage(context, CGRectMake(0, 0, width, height), imageRef);
        CGImageRef imageRefWithoutAlpha = CGBitmapContextCreateImage(context);
        UIImage *imageWithoutAlpha = [UIImage imageWithCGImage:imageRefWithoutAlpha
                                                         scale:image.scale
                                                   orientation:image.imageOrientation];
        
        CGContextRelease(context);
        CGImageRelease(imageRefWithoutAlpha);
        
        return imageWithoutAlpha;
    }
}
```

## 内存优化:下采样
对于图片尺寸远大于`UIImageView`的情况,如果直接加载原图会造成很大的内存占用.此时可以通过`ImageIO`函数`CGImageSourceCreateThumbnailAtIndex`实现
```
- (UIImage *)downsampleImage:(NSURL *)url size:(CGSize)pointSize
{
    NSDictionary *imgSourceOptions = @{
        (NSString *)kCGImageSourceShouldCache:@NO
    };
    CGImageSourceRef imageSource = CGImageSourceCreateWithURL((__bridge CFURLRef)url, (__bridge CFDictionaryRef)imgSourceOptions);
    int maxDimensionInPixels = MAX(pointSize.width, pointSize.height);
    //创建缩略图等比缩放大小
    NSDictionary *downsampleOptions = @{
                                    (NSString *)kCGImageSourceCreateThumbnailFromImageAlways : @YES,
                                    (NSString *)kCGImageSourceShouldCacheImmediately:@YES,
                                    (NSString *)kCGImageSourceThumbnailMaxPixelSize : [NSNumber numberWithInt:maxDimensionInPixels],
                                    (NSString *)kCGImageSourceCreateThumbnailWithTransform : @YES,
                                    };
    
    CGImageRef imageRef = CGImageSourceCreateThumbnailAtIndex(imageSource, 0, (__bridge CFDictionaryRef)downsampleOptions);
    CFRelease(imageSource);
    UIImage *image = [UIImage imageWithCGImage:imageRef];
    CGImageRelease(imageRef);
    return image;
}
```
>上文的缩略图生成过程中，已经对图片进行解码操作，此时的UIImage只是一个CGImage的封装，所以当UIImage赋值给UIImageView时，CALayer可以直接使用CGImage所持有的图像数据

## 与`SDWebImage`的配合使用
在`SDWebImage`中图片异步解码默认是被开启的,具体可以参考`shouldDecompressImages`属性.
在图片被下载完成之后,会自动在子线程中将图片解码,然后将**解码后的图片**按照程序设定存入内存或者磁盘.

下载后图片的转换可以使用`SDWebImageManagerDelegate`的`transform`方法,比如我们要做的降采样或者图片圆角,可以在这个代理方法中进行
```
//该方法在下载完图片之后,会立即对图片进行转换;这个操作是缓存入磁盘和内存之前的
//该代理方法在异步线程执行
- (UIImage *)imageManager:(SDWebImageManager *)imageManager transformDownloadedImage:(UIImage *)image withURL:(NSURL *)imageURL
```
部分源码解读
```
if (downloadedImage  && [self.delegate respondsToSelector:@selector(imageManager:transformDownloadedImage:withURL:)]) {
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
        //获取转换后的图片
        UIImage *transformedImage = [self.delegate imageManager:self transformDownloadedImage:downloadedImage withURL:url];

        //图片存储
        if (transformedImage && finished) {
            BOOL imageWasTransformed = ![transformedImage isEqual:downloadedImage];
            [self.imageCache storeImage:transformedImage recalculateFromImage:imageWasTransformed imageData:(imageWasTransformed ? nil : data) forKey:key toDisk:cacheOnDisk];
        }

        //回调给上层
        dispatch_main_sync_safe(^{
            if (strongOperation && !strongOperation.isCancelled) {
                completedBlock(transformedImage, nil, SDImageCacheTypeNone, finished, url);
            }
        });
    });
}
``` 

## 效果与挑战
1.异步解码图片分担了主线程的压力,效果立竿见影

2.当对于一张 1863 * 4042像素 6M 大小的图片进行下采样的时候,其内存占用只有28.4M,相比使用`imageNamed`直接加载占用的57M,有明显提升.

3.当这一张图片重复显示的时候,下采样每次都会出现一次CPU占用高峰,目前尚不知该如何解决;而使用`imageNamed`加载的图片在重复显示的时候,由于系统对解码后的图片进行了缓存,所以再次显示的时候CPU只有很低的占用.

4.列表加载大量图片的处理技巧
当在一个列表加载大量的图片时,我们可以使用`UITableView`或`UICollectionView`提供的预加载方法来预处理图片
```
- (void)tableView:(UITableView *)tableView prefetchRowsAtIndexPaths:(NSArray<NSIndexPath *> *)indexPaths

- (void)collectionView:(UICollectionView *)collectionView prefetchItemsAtIndexPaths:(NSArray<NSIndexPath *> *)indexPaths
```

使用GCD异步队列来进行异步解码,当图片数量大于CPU核心数时,会出现线程爆炸,即创建大量的线程,为了避免出现这种情况,你应该使用串行队列来操作.

参考链接:

[iOS 图像解码和最佳实践](https://blog.jamchenjun.com/2018/08/22/image-and-graphics-best-practices.html)

[iOS性能优化——图片加载和处理](https://www.jianshu.com/p/7d8a82115060)

[谈谈 iOS 中图片的解压缩](http://blog.leichunfeng.com/blog/2017/02/20/talking-about-the-decompression-of-the-image-in-ios/)

[iOS图片加载速度极限优化—FastImageCache解析](https://blog.cnbang.net/tech/2578/)