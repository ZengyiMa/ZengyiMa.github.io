---
layout: post
title: "Method Swizzling"
author: "Damien"
# catalog: true
tags:
    - iOS
    - Objective-C
--- 



# 介绍
Method Swizzling 是 OC 非常实用的特性，它可以动态的替换2个方法的实现，是面向切面编程的一种手段，举个比较经典的例子：
如果想为 APP 中的每个界面添加访问统计，这个时候可以为每个 VC 的 viewWillAppear 方法添加统计代码，传统的做法是写一个基类，然后让所有的 VC 都继承此基类，从而都有了统计的功能，即使如此也不得不修改原有的类的代码，并且如果是第三方提供的 VC 也不能添加统计的功能。如果是 Method Swizzling 的手段，我们可以在 runtime 的时候动态替换 viewWillAppear 的方法，首先实现一个可以统计的 my_viewWillAppear 方法，之后就可以使用 Method Swizzling 来替换 UIKit 提供的 viewWillAppear 从而实现任意界面添加打点代码，并且不需要修改一行代码。

# 使用
使用 Method Swizzling 一般使用一下的格式

```
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];

        SEL originalSelector = @selector(xxxx:);
        SEL swizzledSelector = @selector(xxx_xxxx:);

        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);

        // When swizzling a class method, use the following:
        // Class class = object_getClass((id)self);
        // ...
        // Method originalMethod = class_getClassMethod(class, originalSelector);
        // Method swizzledMethod = class_getClassMethod(class, swizzledSelector);

        BOOL didAddMethod =
            class_addMethod(class,
                originalSelector,
                method_getImplementation(swizzledMethod),
                method_getTypeEncoding(swizzledMethod));

        if (didAddMethod) {
            class_replaceMethod(class,
                swizzledSelector,
                method_getImplementation(originalMethod),
                method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}
```

我们一起来看看是如何工作的

## 应该在 + load 中使用
+load 是在一个类被初始装载时调用，+initialize 是在应用第一次调用该类的类方法或实例方法前调用的，如果一个类从没被调用，那么它的 initialize 永远不会被调用，所以我们应该在 + load 中 Method Swizzling。

## 用 dispatch_once 来包裹
我们应该始终保证 Method Swizzling 只会进行一次，使用 dispatch_once 来保证唯一性

## 替换

```
Class class = [self class];

        SEL originalSelector = @selector(xxxx:);
        SEL swizzledSelector = @selector(xxx_xxxx:);

        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
```
首先先获取到 method。

```
  BOOL didAddMethod =
            class_addMethod(class,
                originalSelector,
                method_getImplementation(swizzledMethod),
                method_getTypeEncoding(swizzledMethod));
```
判断原方法时候有存在，如果是存在的那么 didAddMethod 将会是 NO，如果没存在则会优先添加。

```
class_replaceMethod(class,
                swizzledSelector,
                method_getImplementation(originalMethod),
                method_getTypeEncoding(originalMethod));
```
如果是没存在的状态，那么 originalSelector 的实现会是 swizzledSelector 的实现，所以我们这边只需要将 swizzledSelector 的实现替换成 originalSelector 实现即可。


```
method_exchangeImplementations(originalMethod, swizzledMethod);
```

如果是已经存在的方法，可以使用 method_exchangeImplementations 来交换2个方法的实现。

# 源码下的 Method Swizzling 

Method Swizzling 主要使用了3个 runtime 方法，分别是 class_addMethod，class_replaceMethod，method_exchangeImplementations
## class_addMethod

此函数的功能是为类动态的添加一个方法，底层调用了 addMethod 方法，并且将最后一个参数设置为 NO。

```
BOOL 
class_addMethod(Class cls, SEL name, IMP imp, const char *types)
{
    if (!cls) return NO;

    rwlock_writer_t lock(runtimeLock);
    return ! addMethod(cls, name, imp, types ?: "", NO);
}

```

在 addMethod 中 最好一个参数表示的是在添加一个已经存在的方法实现情况下时候需要替换这个方法的实现，显然 class_addMethod 将此参数设置为 NO,默认不进行替换。

addMethod 的实现思路为：
首先会寻找 class 中是否有需要添加 sel 的实现（IMP），如果找到，那么会根据 replace 参数来判断时候进行替换，然后将此 IMP 返回。
如果没找到，会为这个类的方法列表中添加一个 method，也就是添加一个方法。

```
static IMP 
addMethod(Class cls, SEL name, IMP imp, const char *types, bool replace)
{
    IMP result = nil;

    runtimeLock.assertWriting();

    assert(types);
    assert(cls->isRealized());

    method_t *m;
    if ((m = getMethodNoSuper_nolock(cls, name))) {
        // already exists
        if (!replace) {
            result = m->imp;
        } else {
            result = _method_setImplementation(cls, m, imp);
        }
    } else {
        // fixme optimize
        method_list_t *newlist;
        newlist = (method_list_t *)calloc(sizeof(*newlist), 1);
        newlist->entsizeAndFlags = 
            (uint32_t)sizeof(method_t) | fixed_up_method_list;
        newlist->count = 1;
        newlist->first.name = name;
        newlist->first.types = strdup(types);
        if (!ignoreSelector(name)) {
            newlist->first.imp = imp;
        } else {
            newlist->first.imp = (IMP)&_objc_ignored_method;
        }

        prepareMethodLists(cls, &newlist, 1, NO, NO);
        cls->data()->methods.attachLists(&newlist, 1);
        flushCaches(cls);

        result = nil;
    }

    return result;
}

```

## class_replaceMethod
class_replaceMethod 底层也是调用了 addMethod 方法，并且将最后一个参数设置为 YES。来表示即使原方法存在，那么就去替换此方法的实现（IMP）

```
IMP 
class_replaceMethod(Class cls, SEL name, IMP imp, const char *types)
{
    if (!cls) return nil;

    rwlock_writer_t lock(runtimeLock);
    return addMethod(cls, name, imp, types ?: "", YES);
}
```

## method_exchangeImplementations 

此方法的功能是交换2个方法的实现（IMP）


```
void method_exchangeImplementations(Method m1, Method m2)
{
    if (!m1  ||  !m2) return;

    rwlock_writer_t lock(runtimeLock);

    if (ignoreSelector(m1->name)  ||  ignoreSelector(m2->name)) {
        // Ignored methods stay ignored. Now they're both ignored.
        m1->imp = (IMP)&_objc_ignored_method;
        m2->imp = (IMP)&_objc_ignored_method;
        return;
    }

    IMP m1_imp = m1->imp;
    m1->imp = m2->imp;
    m2->imp = m1_imp;


    // RR/AWZ updates are slow because class is unknown
    // Cache updates are slow because class is unknown
    // fixme build list of classes whose Methods are known externally?

    flushCaches(nil);

    updateCustomRR_AWZ(nil, m1);
    updateCustomRR_AWZ(nil, m2);
}
```

首先判断方法是否是 ignoreSelector ，ignoreSelector 分别为 retain，release，autorelease 等系统关键方法被交换导致异常。
然后交换2个方法的 imp 指针来达到交换方法的实现

```
  IMP m1_imp = m1->imp;
    m1->imp = m2->imp;
    m2->imp = m1_imp;
```
之后更新缓存和更新一下 RR/AWZ 标志位










