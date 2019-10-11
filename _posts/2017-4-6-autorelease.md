---
layout: post
title: "深入 Autorelease"
author: "Damien"
# catalog: true
tags:
    - iOS
    - Objective-C
--- 

Autorelease 制是 iOS 提供延迟释放对象的一种机制，放弃对象所有权，但又不想对象会给立即释放.这个时候 Autorelease 就能为你解决这个问题，他可以当做寄存处，你可以把对象寄存在它那儿，然后在合适的时间去释放寄存对象。下面让我们深入 
Autorelease。

# 从添加对象开始
在 MRC 时代，可以使用 [xxx autorelease]，来添加一个对象到一个 Autorelease pool 中，什么是  Autorelease pool ？  Autorelease pool 是对象池，统一管理 autorelease 对象的地方。

在 ARC 时代苹果禁用了 autorelease 方法，不能用了，不怕，我们还有 `@autorelease{}`可以使用。

为什么我们要添加 autorelease 对象，我们可以举一个很常见的例子：你需要在一个函数中处理大量零碎的对象，然而函数结束之后才会释放这些对象，这个就可以包裹一个 `@autorelease{}` 可以有效的控制内存的占用。

# AutoreleasePool 结构
在我们写下  `@autorelease{}`的时候，编译器为我们转换成
```
void *context = objc_autoreleasePoolPush();
// 你的代码
objc_autoreleasePoolPop(context);
```
这2个函数在做什么呢?实际都是对`AutoreleasePoolPage `的封装
```
void *objc_autoreleasePoolPush(void) {
    return AutoreleasePoolPage::push();
}

void objc_autoreleasePoolPop(void *ctxt) {
    AutoreleasePoolPage::pop(ctxt);
}
```
AutoreleasePoolPage 对象结构如下图

![AutoreleasePoolPage结构](http://upload-images.jianshu.io/upload_images/809311-fa45561b82906a98.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以注意到`child`和`parent`2个字段，可以看出`AutoreleasePool `是由多个 `AutoreleasePoolPage `组成的双向列表

![双向链表](http://upload-images.jianshu.io/upload_images/809311-f2946217d55cac47.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# AutoreleasePool Push
每个`AutoreleasePoolPage ` 的大小都是 4096 字节，除去 7 字节存储成员变量，其他部分都是用来存放 Autorelease 对象的指针。在 `AutoreleasePool Push`的时候系统会初始化一个`AutoreleasePoolPage `，填充相应的字段，首先填入一个哨兵对象。

哨兵对象定义为：
```
#define POOL_SENTINEL nil
```
为什么要最先添加哨兵对象呢？哨兵对象也是 nil ，在push 的时候添加到栈顶是为了在 Pop （release）的时候，会连续 Pop （release）到第一个哨兵对象。
![13D49DDE-1D53-4620-BAA9-336963D21675.png](http://upload-images.jianshu.io/upload_images/809311-652d4608981c7ebc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

添加完哨兵对象之后就将 next 指针指向下一个添加的 Autorelease 对象。以此加入对象，如果一个`AutoreleasePoolPage `被用慢，那么会开启一个新的`AutoreleasePoolPage `，并且将前一个的child指针指向新的`AutoreleasePoolPage `，添加双向链表节点。

# AutoreleasePool Pop
Pop 操作会以此释放栈顶对象，会一直释放对象到一个哨兵对象，也就是上文说的。
当然 Pop 也支持 Pop 到指定对象。

# Autoreleasepool 与 Runloop 的关系
 Autoreleasepool 处理了 RunLoop 的3个事件
1. Entry(即将进入Loop)，这个时候会调用 _objc_autoreleasePoolPop 来创建一个新的释放池，
2. BeforeWaiting(准备进入休眠) 时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池。
3. Exit(即将退出Loop) 时调用 _objc_autoreleasePoolPop() 来释放自动释放池

看到一些有趣的问题。

# 什么对象自动加入到 autoreleasepool中?
1. 非 alloc/new/copy/mutableCopy 的方法生成的对象如`[NSArray array]`。
2. __weak修饰符 修饰的对象。
3. id的指针或对象的指针在没有显式指定时会被附加上__autorealeasing修饰符

# 子线程默认不会开启 Runloop，那出现 Autorelease 对象如何处理？不手动处理会内存泄漏吗？

子线程如果没有创建 Pool ，但是产生了 Autorelease 对象，就会调用 autoreleaseNoPage 方法。在这个方法中，会自动帮你创建一个 hotpage，也就是默认生成一个 `AutoreleasePoolPage `来添加 autorelease 对象。
