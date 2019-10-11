---
layout: post
title: "MLeaksFinder"
author: "Damien"
# catalog: true
tags:
    - iOS
    - Objective-C
    - 源码阅读
--- 


[TOC]

# 简介
[MLeaksFinder](https://github.com/Zepo/MLeaksFinder) 是WeRead团队开源的一款检测 iOS 内存泄漏的框架，其使用非常简单，只需将文件加入项目中，如果有内存泄漏，3秒后自动弹出 alert 来捕捉循环引用。使得可以在开发快速找到80%内存泄漏，而使用 Xcode Leak 工具更适合大范围的，全部的寻找泄漏点。

# 特性
通过阅读 MLeaksFinder 的介绍可以看出其具有以下几个特性

1. 无侵入性
2. 可以构建泄漏堆栈
3. 有白名单机制
4. 扩展性
5. 其他的一些特殊处理

# 深入源码

## 实现的核心
在 MLeaksFinder 的博客介绍中可以清晰的知道其的工作原理，引用博文所说
> MLeaksFinder 一开始从 UIViewController 入手。我们知道，当一个 UIViewController 被 pop 或 dismiss 后，该 UIViewController 包括它的 view，view 的 subviews 等等将很快被释放（除非你把它设计成单例，或者持有它的强引用，但一般很少这样做）。于是，我们只需在一个 ViewController 被 pop 或 dismiss 一小段时间后，看看该 UIViewController，它的 view，view 的 subviews 等等是否还存在。

> 具体的方法是，为基类 NSObject 添加一个方法 -willDealloc 方法，该方法的作用是，先用一个弱指针指向 self，并在一小段时间(3秒)后，通过这个弱指针调用 -assertNotDealloc，而 -assertNotDealloc 主要作用是直接中断言。这样，当我们认为某个对象应该要被释放了，在释放前调用这个方法，如果3秒后它被释放成功，weakSelf 就指向 nil，不会调用到 -assertNotDealloc 方法，也就不会中断言，如果它没被释放（泄露了），-assertNotDealloc 就会被调用中断言。这样，当一个 UIViewController 被 pop 或 dismiss 时（我们认为它应该要被释放了），我们遍历该 UIViewController 上的所有 view，依次调 -willDealloc，若3秒后没被释放，就会中断言。

总结起来一句话就是，当一个对象3秒之后还没释放，那么指向它的 weak 指针还是存在的，所以可以调用其 runtime 绑定的方法 willDealloc 从而提示内存泄漏。

接下来我们来看看具体实现


## 寻找释放点
论无侵入性的最佳实践还是使用 AOP 面向切面的编程，通过  Method  Swizzling 系统方法来添加额外的功能。
在 MLeaksFinder 中，其 Swizzling 了几个 ViewController 释放方法



```
// UIViewController 的方法

 [self swizzleSEL:@selector(viewDidDisappear:) withSEL:@selector(swizzled_viewDidDisappear:)];
 [self swizzleSEL:@selector(viewWillAppear:) withSEL:@selector(swizzled_viewWillAppear:)];
 [self swizzleSEL:@selector(dismissViewControllerAnimated:completion:) withSEL:@selector(swizzled_dismissViewControllerAnimated:completion:)];
```
通过替换`viewDidDisappear`、`viewWillAppear`、`dismissViewControllerAnimated:completion:` 方法来跟踪一个 modal viewcontroller 的释放


```

//  UINavigationController
[self swizzleSEL:@selector(pushViewController:animated:) withSEL:@selector(swizzled_pushViewController:animated:)];
[self swizzleSEL:@selector(popViewControllerAnimated:) withSEL:@selector(swizzled_popViewControllerAnimated:)];
[self swizzleSEL:@selector(popToViewController:animated:) withSEL:@selector(swizzled_popToViewController:animated:)];
[self swizzleSEL:@selector(popToRootViewControllerAnimated:) withSEL:@selector(swizzled_popToRootViewControllerAnimated:)];
```

而在 UINavigationController 方面，hook 了 `pushViewController:animated:`、 `popViewControllerAnimated:`、`popToViewController:animated:`、`popToRootViewControllerAnimated:`方法来追踪一个 UINavigationController 栈的释放。

然而代码中没发现 `UIPageViewController`、`UISplitViewController`等方法的 hook ，可以自己实现来扩展此。

## 追踪泄漏
`willDealloc`是一个重要的方法。在之前的追踪到一个页面需要释放的时候会调用此方法， 而这个方法做了什么呢？，在 NSObject 的 MemoryLeak 分类里面可以找到这个答案。

```
    NSString *className = NSStringFromClass([self class]);
    if ([[NSObject classNamesInWhiteList] containsObject:className])
        return NO;
    
    NSNumber *senderPtr = objc_getAssociatedObject([UIApplication sharedApplication], kLatestSenderKey);
    if ([senderPtr isEqualToNumber:@((uintptr_t)self)])
        return NO;
    
    __weak id weakSelf = self;
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        __strong id strongSelf = weakSelf;
        [strongSelf assertNotDealloc];
    });
    
    return YES;

```

代码清晰易懂，其原理是首先判断这个 class 是不是在白名单之中，如果是则忽略它， 这也就是上文提过的**白名单机制了**，下面一段代码很有意思，

NSNumber *senderPtr = objc_getAssociatedObject([UIApplication sharedApplication], kLatestSenderKey);

这个方法是干嘛的呢，这就得提到 UIControl 的target-action 机制了，具体可以参考[这篇文章](http://southpeak.github.io/blog/2015/12/13/cocoa-uikit-uicontrol/)。此段代码的意义在与，如果当前的对象在发送 action 则忽略它（因为 willDealloc 总会先于他们调用）， 之后设置一个 weak 指针，在调用 dispatch_after 在2秒后调用 assertNotDealloc 方法，如果还没释放，那么会进入这个方法，如果已经释放了，那么这个方法是进不去的。

##  报告泄漏
当进入 assertNotDealloc 方法，很不幸，这个对象极有可能是泄漏了，这个时候应该做的报告此处发生了泄漏

```
- (void)assertNotDealloc {
    if ([MLeakedObjectProxy isAnyObjectLeakedAtPtrs:[self parentPtrs]]) {
        return;
    }
    [MLeakedObjectProxy addLeakedObject:self];
    
    NSString *className = NSStringFromClass([self class]);
    NSLog(@"Possibly Memory Leak.\nIn case that %@ should not be dealloced, override -willDealloc in %@ by returning NO.\nView-ViewController stack: %@", className, className, [self viewStack]);
}
```
MLeaksFinder 是如何报告的呢，我们一步步追踪，
首先

```
    if ([MLeakedObjectProxy isAnyObjectLeakedAtPtrs:[self parentPtrs]]) {
        return;
    }
```

这个是什么东西，

```
@interface MLeakedObjectProxy : NSObject

+ (BOOL)isAnyObjectLeakedAtPtrs:(NSSet *)ptrs;
+ (void)addLeakedObject:(id)object;

@end

```

可以看出这个是泄漏对象的代理，这个对象只有2个类方法，分别是做什么的呢，先来看 isAnyObjectLeakedAtPtrs 这个方法

```

 + (BOOL)isAnyObjectLeakedAtPtrs:(NSSet *)ptrs {
    NSAssert([NSThread isMainThread], @"Must be in main thread.");
    
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        leakedObjectPtrs = [[NSMutableSet alloc] init];
    });
    
    if (!ptrs.count) {
        return NO;
    }
    if ([leakedObjectPtrs intersectsSet:ptrs]) {
        return YES;
    } else {
        return NO;
    }
}

```

结合上文中的 assertNotDealloc 方法可以看出，这个方法主要是判断当前这个对象时候已经添加到了泄漏对象名单，如果是，那么就不在添加了。
还有一个 addLeakedObject 方法，看名字不难看出是将泄漏对象添加到泄漏对象名单，以下下是其具体实现

```
+ (void)addLeakedObject:(id)object {
    NSAssert([NSThread isMainThread], @"Must be in main thread.");
    
    MLeakedObjectProxy *proxy = [[MLeakedObjectProxy alloc] init];
    proxy.object = object;
    proxy.objectPtr = @((uintptr_t)object);
    proxy.viewStack = [object viewStack];
    static const void *const kLeakedObjectProxyKey = &kLeakedObjectProxyKey;
    objc_setAssociatedObject(object, kLeakedObjectProxyKey, proxy, OBJC_ASSOCIATION_RETAIN);
    
    [leakedObjectPtrs addObject:proxy.objectPtr];
    
#if _INTERNAL_MLF_RC_ENABLED
    [MLeaksMessenger alertWithTitle:@"Memory Leak"
                            message:[NSString stringWithFormat:@"%@", proxy.viewStack]
                           delegate:proxy
              additionalButtonTitle:@"Retain Cycle"];
#else
    [MLeaksMessenger alertWithTitle:@"Memory Leak"
                            message:[NSString stringWithFormat:@"%@", proxy.viewStack]];
#endif
}
```

可以看出，构造了一个 MLeakedObjectProxy 对象，并将其加入到 leakedObjectPtrs 集合中，弹出 alert 框，然后需要找到引用环，将此对象指针传给 FBRetainCycleDetector 来确定循环引用发生在哪里（这块功能是0.2版本新增的）。之后在屏幕上打印出堆栈信息，看到这里，还有一个疑问：MLeaksFinder 是如何构建堆栈？

## 构建堆栈信息
我们可以注意到，在 UIViewController 的分类中有这样一段代码

```
- (BOOL)willDealloc {
    if (![super willDealloc]) {
        return NO;
    }
    
    [self willReleaseChildren:self.childViewControllers];
    [self willReleaseChild:self.presentedViewController];
    
    if (self.isViewLoaded) {
        [self willReleaseChild:self.view];
    }
    
    return YES;
}
```
其中`willReleaseChildren`、`willReleaseChild`，就是构造堆栈信息的秘密所在，我们可以看到 NSObject 分类中的实现方法

```
- (void)willReleaseChild:(id)child {
    if (!child) {
        return;
    }
    
    [self willReleaseChildren:@[ child ]];
}

- (void)willReleaseChildren:(NSArray *)children {
    NSArray *viewStack = [self viewStack];
    NSSet *parentPtrs = [self parentPtrs];
    for (id child in children) {
        NSString *className = NSStringFromClass([child class]);
        [child setViewStack:[viewStack arrayByAddingObject:className]];
        [child setParentPtrs:[parentPtrs setByAddingObject:@((uintptr_t)child)]];
        [child willDealloc];
    }
}

- (NSArray *)viewStack {
    NSArray *viewStack = objc_getAssociatedObject(self, kViewStackKey);
    if (viewStack) {
        return viewStack;
    }
    
    NSString *className = NSStringFromClass([self class]);
    return @[ className ];
}

- (void)setViewStack:(NSArray *)viewStack {
    objc_setAssociatedObject(self, kViewStackKey, viewStack, OBJC_ASSOCIATION_RETAIN);
}

```
willReleaseChildren 和 willReleaseChild 方法的作用是向这个对象之中的子对象调用释放的方法，如 view 的 subviews， UINavigationController 的 viewcontrollers 等等的子对象。构造堆栈信息的原理就是，递归遍历子对象，然后将父对象 class name 加上子对象 class name，一步步构造出一个 view stack。出现泄漏则直接打印此对象的 view stack 即可。

## 其他
在 UIViewController 侧滑的时候释放方法需要做特殊的处理,在 MLeaksFinder 中添加了 kHasBeenPoppedKey 属性来判断是否释放代码如下


```

//设置侧滑的key
- (UIViewController *)swizzled_popViewControllerAnimated:(BOOL)animated {
    UIViewController *poppedViewController = [self swizzled_popViewControllerAnimated:animated];
    
    if (!poppedViewController) {
        return nil;
    }
    
    // Detail VC in UISplitViewController is not dealloced until another detail VC is shown
 ...
    
    // VC is not dealloced until disappear when popped using a left-edge swipe gesture
    extern const void *const kHasBeenPoppedKey;
    objc_setAssociatedObject(poppedViewController, kHasBeenPoppedKey, @(YES), OBJC_ASSOCIATION_RETAIN);
    
    return poppedViewController;
}

//发现是侧滑则直接调用willDealloc方法
- (void)swizzled_viewDidDisappear:(BOOL)animated {
    [self swizzled_viewDidDisappear:animated];
    
    if ([objc_getAssociatedObject(self, kHasBeenPoppedKey) boolValue]) {
        [self willDealloc];
    }
}

```
# 总结
总得来说 MLeaksFinder 是一个质量很高的库，在实用性和便利性上做到了完美结合，日常开发中也为我们的项目寻找到了泄漏点，值得一用，至于 FBRetainCycleDetector 如何检测循环引用环，那得发一篇文章的篇幅来介绍了