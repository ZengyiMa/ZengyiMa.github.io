---
layout: post
title: "iOS 的事件机制"
author: "Damien"
# catalog: true
tags:
    - iOS
    - Objective-C
--- 

# 响应链
iOS 大多数的事件分发都是依赖 UIResponder 响应链来完成，响应链是由一系列链接在一起的响应者组成的。一般情况下，一条响应链开始于第一响应者，结束于application对象。如果一个响应者不能处理事件，则会将事件沿着响应链传到下一响应者。那么当我点击屏幕，系统发生了什么呢？

我们以如下的视图层级为例：

![](http://upload-images.jianshu.io/upload_images/144142-3afc2c85792ade41.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## hit-test
在点击之后，系统会优先找出响应的视图，此过程叫做 hit-test

关于 hit-test 有 2 个方法与其有关
```
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event; 
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event;
```

当你点击到 View E 的时候系统做发生一些什么呢，

1.  系统底层发出一个 event
2. 系统分发事件到我们 App 的 UIApplication
3. UIApplication 会对当前的 KeyWindow 调用 hitTest 方法来确定触摸的位置时候在 Window 内，hitTest 方法会调用 pointInSide 方法来确认一个点是否包含在我们 Window 视图之类。
4. UIWindow 会对其子视图，也就是 View A 调用 hitTest ，hitTest 如步骤 3，调用 pointInSide 来确定点时候爱自身之类，这个时候返回 NO,  因为 E 不在 A 的里面。
5. ViewC 对其 自身做 hitTest 和 pointInSide ，返回 YES, 因为 E 在 C 内。
6. View D 对其 自身做 hitTest 和 pointInSide ，返回 NO, 因为 E 不在 D 内。
7. View E 对其 自身做 hitTest 和 pointInSide ，返回 YES
8. 由于 View E 是 hitTest 的最终点，没有其他子视图，于是结束 hitTest 的流程，返回自身，结束递归回调。

像上面的步骤，我们将寻找的过程称之为 hit-Testing， 这个被找到的 View 称之为 hit-Test View 。

找到 hit-Test View 之后我们便可以确定响应的对象，

## touchesEvent

确定了第一响应者之后， touchesBegan，touchesMoved，touchesEnded 方法会根据情况依次被调用。

关于 touch 的方法有 4 个分别是：
```
open func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?)
open func touchesMoved(_ touches: Set<UITouch>, with event: UIEvent?)
open func touchesEnded(_ touches: Set<UITouch>, with event: UIEvent?)
open func touchesCancelled(_ touches: Set<UITouch>, with event: UIEvent?)

```


## 事件传递
最有机会处理事件的对象是hit-test视图或第一响应者。如果这两者都不能处理事件，UIKit就会将事件传递到响应链中的下一个响应者。每一个响应者确定其是否要处理事件或者是通过nextResponder方法将其传递给下一个响应者。这一过程一直持续到找到能处理事件的响应者对象或者最终没有找到响应者。
![事件传递的流程](https://developer.apple.com/library/content/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/Art/iOS_responder_chain_2x.png)

当系统检测到一个事件时，将其传递给初始对象，这个对象通常是一个视图。然后，会按以下路径来处理事件(我们以左图为例)：

初始视图(initial view)尝试处理事件。如果它不能处理事件，则将事件传递给其父视图。
初始视图的父视图(superview)尝试处理事件。如果这个父视图还不能处理事件，则继续将视图传递给上层视图。
上层视图(topmost view)会尝试处理事件。如果这个上层视图还是不能处理事件，则将事件传递给视图所在的视图控制器。
视图控制器会尝试处理事件。如果这个视图控制器不能处理事件，则将事件传递给窗口(window)对象。
窗口(window)对象尝试处理事件。如果不能处理，则将事件传递给单例app对象。
如果app对象不能处理事件，则丢弃这个事件。
从上面可以看到，视图、视图控制器、窗口对象和app对象都能处理事件。


## 手势识别器对响应链的影响
![手势和UIView 响应链的区别](https://developer.apple.com/library/content/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/Art/path_of_touches_2x.png)
手势相对于 UIView 的响应链拥有更高的级别，在事件分发的时候会优先给手势处理，是否进行正常的响应链流程则有手势识别器来决定。

```
cancelsTouchesInView
```
此属性可以中断响应链的流程，响应流程会取消，使 `touchesEnded` 中断，而 `touchesCancelled` 被调用。

```
delaysTouchesBegan
```
此属性可以延迟响应链的开始的流程，使 `touchesBegan ` 延后导致响应链被取消。

```
delaysTouchesEnded
```
此属性可以延迟响应链的开始的末尾，使 `touchesEnded ` 延后，从而响应链不能完整的结束。

### 手势的代理
```
- (BOOL)gestureRecognizerShouldBegin:(UIGestureRecognizer *)gestureRecognizer;
```
手势识别以及后续事件是否开始激活，返回 NO 则终止了手势。

```
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRecognizeSimultaneouslyWithGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer;
```
这个方法指示出 gestureRecognizer 是否可以和其他的 gestureRecognizer 一起处理事件。

```
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRequireFailureOfGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer NS_AVAILABLE_IOS(7_0);
```
返回 YES 时，当 gestureRecognizer 和 otherGestureRecognizer 同时响应时候，gestureRecognizer 失效

```
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRequireFailureOfGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer NS_AVAILABLE_IOS(7_0);
```
返回 YES 时，当 gestureRecognizer 和 otherGestureRecognizer 同时响应时候，otherGestureRecognizer 失效


## UIControl 及其子类的特殊化
上文提到 手势识别器可以拦截一个响应链，那么我的 UIButton 为什么可以正常工作呢？
文档中有注明
>When a control-specific event occurs, the control calls any associated action methods right away. Action methods are dispatched through the current UIApplication object, which finds an appropriate object to handle the message, following the responder chain if needed.

一个 UIControl 及其子类 、方法，其响应事件的方式不同于普通的UIView，它们的事件处理由UIApplication直接分发，和普通的UIView完全不同。所以自然手势识别器无法影响。