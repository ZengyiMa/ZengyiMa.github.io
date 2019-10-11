---
layout: post
title: "Block"
author: "Damien"
# catalog: true
tags:
    - iOS
    - Objective-C
    - 源码阅读
--- 

[TOC]

MBProgressHUD 是iOS 第三方Hub 框控件。

# 结构
## 控件组成
在MBProgressHUD（以下简称MB）中hub 的结构是由几部分组成的

* **indicatorView** 展示菊花，进度条等样式的视图。根据不同mode来创建不同的View。
* **backgroundView**背景View，处理暗色，或者模糊样式，对应类**MBBackgroundView**
* 
```
typedef NS_ENUM(NSInteger, MBProgressHUDBackgroundStyle) {
    MBProgressHUDBackgroundStyleSolidColor,
    MBProgressHUDBackgroundStyleBlur
};
```

* **label** 加载框下方的描述信息
* **detailsLabel** 描述信息下面的详细描述信息
* **actionButton** 在提供一个操作的button

## 样式
在MBProgressHUD（以下简称MB）的头文件中描述了几种样式

```
typedef NS_ENUM(NSInteger, MBProgressHUDMode) {
    /// 无限菊花样式.
    MBProgressHUDModeIndeterminate,
    /// 圆形进度条样式.
    MBProgressHUDModeDeterminate,
    /// 横向滚动进度条样式.
    MBProgressHUDModeDeterminateHorizontalBar,
    /// 圆形进度条样式.和第二个有点不同
    MBProgressHUDModeAnnularDeterminate,
    /// 自定义View的样式
    MBProgressHUDModeCustomView,
    /// 只有文字的样式.
    MBProgressHUDModeText
};
```
MB 通过MBProgressHUDMode来创建不同展示Hub 

```
- (void)updateIndicators {
    UIView *indicator = self.indicator;
    BOOL isActivityIndicator = [indicator isKindOfClass:[UIActivityIndicatorView class]];
    BOOL isRoundIndicator = [indicator isKindOfClass:[MBRoundProgressView class]];

    MBProgressHUDMode mode = self.mode;
    if (mode == MBProgressHUDModeIndeterminate) {
        if (!isActivityIndicator) {
            // Update to indeterminate indicator
            [indicator removeFromSuperview];
            indicator = [[UIActivityIndicatorView alloc] initWithActivityIndicatorStyle:UIActivityIndicatorViewStyleWhiteLarge];
            [(UIActivityIndicatorView *)indicator startAnimating];
            [self.bezelView addSubview:indicator];
        }
    }
    else if (mode == MBProgressHUDModeDeterminateHorizontalBar) {
        // Update to bar determinate indicator
        [indicator removeFromSuperview];
        indicator = [[MBBarProgressView alloc] init];
        [self.bezelView addSubview:indicator];
    }
    else if (mode == MBProgressHUDModeDeterminate || mode == MBProgressHUDModeAnnularDeterminate) {
        if (!isRoundIndicator) {
            // Update to determinante indicator
            [indicator removeFromSuperview];
            indicator = [[MBRoundProgressView alloc] init];
            [self.bezelView addSubview:indicator];
        }
        if (mode == MBProgressHUDModeAnnularDeterminate) {
            [(MBRoundProgressView *)indicator setAnnular:YES];
        }
    } 
    else if (mode == MBProgressHUDModeCustomView && self.customView != indicator) {
        // Update custom view indicator
        [indicator removeFromSuperview];
        indicator = self.customView;
        [self.bezelView addSubview:indicator];
    }
    else if (mode == MBProgressHUDModeText) {
        [indicator removeFromSuperview];
        indicator = nil;
    }
    indicator.translatesAutoresizingMaskIntoConstraints = NO;
    self.indicator = indicator;

    if ([indicator respondsToSelector:@selector(setProgress:)]) {
        [(id)indicator setValue:@(self.progress) forKey:@"progress"];
    }

    [indicator setContentCompressionResistancePriority:998.f forAxis:UILayoutConstraintAxisHorizontal];
    [indicator setContentCompressionResistancePriority:998.f forAxis:UILayoutConstraintAxisVertical];

    [self updateViewsForColor:self.contentColor];
    [self setNeedsUpdateConstraints];
}
```
# 具体实现
MB 的实现比较朴实，没有过多花哨的技巧，简单实用。
展示的流程大概是这样的

1. 通过`initWithWindow`和`initWithView`来创建Hub ，这里的Window和View不是MB 来添加的父视图而是算确定frame的视图
2. 通过`showHUDAddedTo`来将Hub 添加到指定的父视图。
3. 调用`showAnimated`通过指定动画来展示出Hub
4. 实用`hideAnimated`来指定动画消失

# 其他
MB 通过通知`UIApplicationDidChangeStatusBarOrientationNotification`来处理屏幕转屏事件

```
- (void)registerForNotifications {
#if !TARGET_OS_TV
    NSNotificationCenter *nc = [NSNotificationCenter defaultCenter];

    [nc addObserver:self selector:@selector(statusBarOrientationDidChange:)
               name:UIApplicationDidChangeStatusBarOrientationNotification object:nil];
#endif
}
```

MB 的MBProgressHUDModeDeterminateHorizontalBar模式是通过drawrect 来绘制的，对应类是**MBBarProgressView** ，`determinanteMode`对应类**MBRoundProgressView**。

# 总结
**优点**
1. MB 的View层级灵活，可以放到不同的View 或者Window 中，而SVPoregress却不行
2. MB 的CustomView可以自定义视图，相对比SVProgress定制性要高

**缺点**
1、 MB 的动画用的是drawrect 而非CALayer 相对没那么高效

