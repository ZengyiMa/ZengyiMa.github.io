---
layout: post
title: "SVProgressHUD"
author: "Damien"
# catalog: true
tags:
    - iOS
    - 源码阅读
--- 

[TOC]
## 简介

[SVProgressHUB](https://github.com/SVProgressHUD/SVProgressHUD)是iOS上的的一款loading轻量级美观的加载框。
首先来看下结构目录
![](http://upload-images.jianshu.io/upload_images/809311-c039fd2e1396b713.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**SVIndefiniteAnimatedView** 是无限旋转的菊花视图
**SVProgressAnimatedView** 是进度条视图
**SVRadialGradientLayer** 是渐变模式下渐变的背景Layer

## 使用

### 显示
SVProgressHUD提供了显示加载图便利方法，而且几乎全是类方法，在开发中能方便的使用。如

* showWithStatus 显示加载，并显示文本
* showProgress		显示加载，并展示当前进度条
* showInfoWithStatus 显示使用Info状态的图片的加载框来替代菊花或者进度条
* showSuccessWithStatus 显示success状态的图片（一个勾）的加载框来替代菊花或者进度条
* showErrorWithStatus 显示error状态的图片（一个x）的加载框来替代菊花或者进度条

除此之外你也可以设置自己的图片，使用`+ (void)showImage:(UIImage*)image status:(NSString*)status;
`

### 消失
SVProgressHUD提供了几种方式来显示一个Loding框

* dismiss 直接关闭一个加载Hub
* dismissWithDelay 延迟关闭一个Hub
* dismissWithCompletion 关闭Hub带一个完成的Block
* dismissWithDelay:completion 延迟关闭Hub并带一个完成的Block

可以看出SVProgressHUD提供了简单明了的API，使用便利的类方法让我们快速在代码中使用

## 深入源码
###显示过程

SVProgressHUD 比较简单易懂
在众多的类方法下面，也实现相应的实例方法

```
//简单版本
- (void)setStatus:(NSString*)status;
- (void)setFadeOutTimer:(NSTimer*)timer;

- (void)registerNotifications;
- (NSDictionary*)notificationUserInfo;

- (void)positionHUD:(NSNotification*)notification;
- (void)moveToPoint:(CGPoint)newCenter rotateAngle:(CGFloat)angle;

- (void)overlayViewDidReceiveTouchEvent:(id)sender forEvent:(UIEvent*)event;

- (void)showProgress:(float)progress status:(NSString*)status;
- (void)showImage:(UIImage*)image status:(NSString*)status duration:(NSTimeInterval)duration;
- (void)showStatus:(NSString*)status;

- (void)dismiss;
- (void)dismissWithDelay:(NSTimeInterval)delay completion:(SVProgressHUDDismissCompletion)completion;
@end

```


如何将Hub展示到屏幕上，首先我们从showWithStatus方法往里看。
内部实现

```
+ (void)showWithStatus:(NSString*)status {
    [self sharedView];
    [self showProgress:SVProgressHUDUndefinedProgress status:status];
}
```
在内部实现了一个单例，这也不难理解为什么可以在内方法中快速的显示一个Hub

```
+ (SVProgressHUD*)sharedView {
    static dispatch_once_t once;
    
    static SVProgressHUD *sharedView;
#if !defined(SV_APP_EXTENSIONS)
    dispatch_once(&once, ^{ sharedView = [[self alloc] initWithFrame:[[[UIApplication sharedApplication] delegate] window].bounds]; });
#else
    dispatch_once(&once, ^{ sharedView = [[self alloc] initWithFrame:[[UIScreen mainScreen] bounds]]; });
#endif
    return sharedView;
}

```
之后使用这个单例视图来完成Hub的显示，那么showProgress这个方法又做什么呢？首先我们看下简化版代码

```
- (void)showProgress:(float)progress status:(NSString*)status {
    __weak SVProgressHUD *weakSelf = self;
    [[NSOperationQueue mainQueue] addOperationWithBlock:^{
        __strong SVProgressHUD *strongSelf = weakSelf;
        if(strongSelf){
             //更新视图层级，将SV放到当前window最前面
            [strongSelf updateViewHierachy];
            //判断如果是进度条，使用进度条View
            if(progress >= 0) {                
                // Add ring to HUD and set progress
                [strongSelf.hudView addSubview:strongSelf.ringView];
                [strongSelf.hudView addSubview:strongSelf.backgroundRingView];
                strongSelf.ringView.strokeEnd = progress;
            } else {
                    
                
                // 使用无限旋转的菊花模式
                [strongSelf.hudView addSubview:strongSelf.indefiniteAnimatedView];
            }
            // Show
            [strongSelf showStatus:status];
        }
    }];
}
```

可以看出，SV为了将显示操作放到主线程使用了`[NSOperationQueue mainQueue] `来确保在主线程操作UI，
在`updateViewHierachy`方法中，SV会寻找提供Hub显示的Window，并且将自己添加到上面。使用手法为:

```
 NSEnumerator *frontToBackWindows = [UIApplication.sharedApplication.windows reverseObjectEnumerator];
        for (UIWindow *window in frontToBackWindows) {
            BOOL windowOnMainScreen = window.screen == UIScreen.mainScreen;
            BOOL windowIsVisible = !window.hidden && window.alpha > 0;
            BOOL windowLevelNormal = window.windowLevel == UIWindowLevelNormal;
            
            if(windowOnMainScreen && windowIsVisible && windowLevelNormal) {
                [window addSubview:self.overlayView];
                break;
            }
        }
```
找出当前的windowLevelNormal,并且是显示的UI，将自己显示在其之上，但是也有点缺点，就是如果需要在自定义的window上面显示Hub将无能为力。
找到window之后，生成菊花或者进度条View，然后设置label，计算出大小，然后完成显示

### 动画的绘制
SVIndefiniteAnimatedView是无限旋转的View。其中使用CASharpLayer和LayerMask来完成一个旋转动画，使用一张渐变的图片，设置其mask属性，然后加上旋转动画，来形成一个漂亮的加载动画
动画绘制代码

```  CGPoint arcCenter = CGPointMake(self.radius+self.strokeThickness/2+5, self.radius+self.strokeThickness/2+5);
        UIBezierPath* smoothedPath = [UIBezierPath bezierPathWithArcCenter:arcCenter radius:self.radius startAngle:(CGFloat) (M_PI*3/2) endAngle:(CGFloat) (M_PI/2+M_PI*5) clockwise:YES];
        
        _indefiniteAnimatedLayer = [CAShapeLayer layer];
        _indefiniteAnimatedLayer.contentsScale = [[UIScreen mainScreen] scale];
        _indefiniteAnimatedLayer.frame = CGRectMake(0.0f, 0.0f, arcCenter.x*2, arcCenter.y*2);
        _indefiniteAnimatedLayer.fillColor = [UIColor clearColor].CGColor;
        _indefiniteAnimatedLayer.strokeColor = self.strokeColor.CGColor;
        _indefiniteAnimatedLayer.lineWidth = self.strokeThickness;
        _indefiniteAnimatedLayer.lineCap = kCALineCapRound;
        _indefiniteAnimatedLayer.lineJoin = kCALineJoinBevel;
        _indefiniteAnimatedLayer.path = smoothedPath.CGPath;
        
        CALayer *maskLayer = [CALayer layer];
        
        NSBundle *bundle = [NSBundle bundleForClass:[SVProgressHUD class]];
        NSURL *url = [bundle URLForResource:@"SVProgressHUD" withExtension:@"bundle"];
        NSBundle *imageBundle = [NSBundle bundleWithURL:url];
        
        NSString *path = [imageBundle pathForResource:@"angle-mask" ofType:@"png"];
        
        maskLayer.contents = (__bridge id)[[UIImage imageWithContentsOfFile:path] CGImage];
        maskLayer.frame = _indefiniteAnimatedLayer.bounds;
        _indefiniteAnimatedLayer.mask = maskLayer;
        
        NSTimeInterval animationDuration = 1;
        CAMediaTimingFunction *linearCurve = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionLinear];
        
        CABasicAnimation *animation = [CABasicAnimation animationWithKeyPath:@"transform.rotation"];
        animation.fromValue = (id) 0;
        animation.toValue = @(M_PI*2);
        animation.duration = animationDuration;
        animation.timingFunction = linearCurve;
        animation.removedOnCompletion = NO;
        animation.repeatCount = INFINITY;
        animation.fillMode = kCAFillModeForwards;
        animation.autoreverses = NO;
        [_indefiniteAnimatedLayer.mask addAnimation:animation forKey:@"rotate"];
        
        CAAnimationGroup *animationGroup = [CAAnimationGroup animation];
        animationGroup.duration = animationDuration;
        animationGroup.repeatCount = INFINITY;
        animationGroup.removedOnCompletion = NO;
        animationGroup.timingFunction = linearCurve;
        
        CABasicAnimation *strokeStartAnimation = [CABasicAnimation animationWithKeyPath:@"strokeStart"];
        strokeStartAnimation.fromValue = @0.015;
        strokeStartAnimation.toValue = @0.515;
        
        CABasicAnimation *strokeEndAnimation = [CABasicAnimation animationWithKeyPath:@"strokeEnd"];
        strokeEndAnimation.fromValue = @0.485;
        strokeEndAnimation.toValue = @0.985;
        
        animationGroup.animations = @[strokeStartAnimation, strokeEndAnimation];
        [_indefiniteAnimatedLayer addAnimation:animationGroup forKey:@"progress"];
```

圆环动画比较简单，原理是使用CAShareLayer，并且设置的endPath来完成一个由progress驱动的进度条动画

```
 CGPoint arcCenter = CGPointMake(self.radius+self.strokeThickness/2+5, self.radius+self.strokeThickness/2+5);
        UIBezierPath* smoothedPath = [UIBezierPath bezierPathWithArcCenter:arcCenter radius:self.radius startAngle:(CGFloat)-M_PI_2 endAngle:(CGFloat) (M_PI + M_PI_2) clockwise:YES];
        
        _ringAnimatedLayer = [CAShapeLayer layer];
        _ringAnimatedLayer.contentsScale = [[UIScreen mainScreen] scale];
        _ringAnimatedLayer.frame = CGRectMake(0.0f, 0.0f, arcCenter.x*2, arcCenter.y*2);
        _ringAnimatedLayer.fillColor = [UIColor clearColor].CGColor;
        _ringAnimatedLayer.strokeColor = self.strokeColor.CGColor;
        _ringAnimatedLayer.lineWidth = self.strokeThickness;
        _ringAnimatedLayer.lineCap = kCALineCapRound;
        _ringAnimatedLayer.lineJoin = kCALineJoinBevel;
        _ringAnimatedLayer.path = smoothedPath.CGPath;

```

### 消失过程
消失调用的是`dismissWithDelay:completion`方法
消失过程和显示过程相反，大致是

* 在mainQueue中添加一个Block的operation（在主线程更改UI）
* 将自己从SuperView中删除，
* 如果指定了延迟，那么设置一个timer
* 在消失中如果指定动画，那么加一个fade的动画来进行消失

## 小结

1. SV将View添加到Window上
2. SV使用NSOperationQueue 来确保自己在主线程修改UI
3. SV的动画使用layer来绘制

优点：API简单明了，调用方便
缺点：默认在APP的UIWindow，不好定制


