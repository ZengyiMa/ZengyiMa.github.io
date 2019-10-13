---
layout: post
title: "YYImage之YYSpriteSheetImage"
author: "Damien"
# catalog: true
tags:
    - iOS
    - 源码阅读
--- 


SpriteSheetImage 从一张大图片上面提取多个小图片来组成一个动画，将多个小图片集成到大图片的好处是可以节省多张图片占据的容量，优化安装包大小。

YYSpriteSheetImage 初始化方法只有一个

```
- (nullable instancetype)initWithSpriteSheetImage:(UIImage *)image
                                     contentRects:(NSArray<NSValue *> *)contentRects
                                   frameDurations:(NSArray<NSNumber *> *)frameDurations
                                        loopCount:(NSUInteger)loopCount;
```
需要传入一张完整的大图，每张小图的 rect ，动画时间，动画循环次数。


```
- (instancetype)initWithSpriteSheetImage:(UIImage *)image
                            contentRects:(NSArray *)contentRects
                          frameDurations:(NSArray *)frameDurations
                               loopCount:(NSUInteger)loopCount {
    if (!image.CGImage) return nil;
    if (contentRects.count < 1 || frameDurations.count < 1) return nil;
    if (contentRects.count != frameDurations.count) return nil;
    
    self = [super initWithCGImage:image.CGImage scale:image.scale orientation:image.imageOrientation];
    if (!self) return nil;
    
    _contentRects = contentRects.copy;
    _frameDurations = frameDurations.copy;
    _loopCount = loopCount;
    return self;
}
```

在内部只是持有这个图片的 image 对象，而在展示的时候才会去截取相应的 rect 的图片。

在来看如何实现 YYAnimatedImage 的协议来播放动画。

返回帧数

```
- (NSUInteger)animatedImageFrameCount {
    return _contentRects.count;
}
```

返回循环次数

```
- (NSUInteger)animatedImageLoopCount {
    return _loopCount;
}
```

返回每一帧的图片

```
- (UIImage *)animatedImageFrameAtIndex:(NSUInteger)index {
    return self;
}
```

每一帧播放的时间

```
- (NSTimeInterval)animatedImageDurationAtIndex:(NSUInteger)index {
    if (index >= _frameDurations.count) return 0;
    return ((NSNumber *)_frameDurations[index]).doubleValue;
}
```

这个方法与其他不同，它返回每一帧的 rect，也就是每一帧图片显示的范围，配合 animatedImageFrameAtIndex 一起来组成 SpriteSheet 

```
- (CGRect)animatedImageContentsRectAtIndex:(NSUInteger)index {
    if (index >= _contentRects.count) return CGRectZero;
    return ((NSValue *)_contentRects[index]).CGRectValue;
}
```