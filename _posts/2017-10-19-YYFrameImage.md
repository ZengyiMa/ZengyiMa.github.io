---
layout: post
title: "YYImage之YYFrameImage"
author: "Damien"
# catalog: true
tags:
    - iOS
    - 源码阅读
--- 



YYFrameImage 代表是一组图片组合而成的动态图片对象。
首先我们来看初始化方法。
YYFrameImage 有2个初始化方法分别是

```
- (instancetype)initWithImagePaths:(NSArray *)paths frameDurations:(NSArray *)frameDurations loopCount:(NSUInteger)loopCount

- (instancetype)initWithImageDataArray:(NSArray *)dataArray frameDurations:(NSArray *)frameDurations loopCount:(NSUInteger)loopCount

```

构造简单，用过一组的图片的 path 或者 data 来把多个图片封装在一起，达在 YYAnimatedImageView 中播放的目的。

其在内部也维护了几个成员变量

```
NSUInteger _loopCount;
NSUInteger _oneFrameBytes;
NSArray *_imagePaths;
NSArray *_imageDatas;
NSArray *_frameDurations;
```
分别是循环次数，多个图片 path或者多个图片 data，还有 每一帧播放时间的数组。

为了在 YYAnimatedImageView 播放，需要实现 YYAnimatedImage 协议，我们来看下多个图片如何组合。

首先告诉 YYAnimatedImageView 有多少帧可以播放。

```
- (NSUInteger)animatedImageFrameCount {
    if (_imagePaths) {
        return _imagePaths.count;
    } else if (_imageDatas) {
        return _imageDatas.count;
    } else {
        return 1;
    }
}

```

之后提供循环的次数


```
- (NSUInteger)animatedImageLoopCount {
    return _loopCount;
}
```
提供每一帧的的大小，为了图片优化

```
- (NSUInteger)animatedImageBytesPerFrame {
    return _oneFrameBytes;
}
```

返回每一帧的图片

```
- (UIImage *)animatedImageFrameAtIndex:(NSUInteger)index {
    if (_imagePaths) {
        if (index >= _imagePaths.count) return nil;
        NSString *path = _imagePaths[index];
        CGFloat scale = _NSStringPathScale(path);
        NSData *data = [NSData dataWithContentsOfFile:path];
        return [[UIImage imageWithData:data scale:scale] yy_imageByDecoded];
    } else if (_imageDatas) {
        if (index >= _imageDatas.count) return nil;
        NSData *data = _imageDatas[index];
        return [[UIImage imageWithData:data scale:[UIScreen mainScreen].scale] yy_imageByDecoded];
    } else {
        return index == 0 ? self : nil;
    }
}
```

提供每一帧的显示时间

```
- (NSTimeInterval)animatedImageDurationAtIndex:(NSUInteger)index {
    if (index >= _frameDurations.count) return 0;
    NSNumber *num = _frameDurations[index];
    return [num doubleValue];
}
```

好了，frameImage 实现比较简单，代码也清晰易懂。更多细节可以自己去挖掘哦。