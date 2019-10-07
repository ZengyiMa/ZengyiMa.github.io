---
layout: post
title: "Objective-C 高级编程读书笔记"
author: "Damien"
# catalog: true
tags:
    - iOS
    - Objective-C
--- 

#  自动引用计数

## ARC
自动引用计数 ARC ：是指内存管理中对引用计数采取自动计数的计数。

苹果文档
> ARC 是让编译器来进行内存管理。设置了 ARC 为有效状态，就无需再次键入 retain 或者 release 代码。降低了程序崩溃，内存泄漏等风险的风险的同时，很大程度减少了开发程序的工作量。编译器完全清楚目标对象，并能立刻释放那些不再使用的对象。

## 内存管理

* 自己生成的对象，自己所持有
* 非自己生成的对象，自己也可以持有
* 不再需要自己持有的对象时释放
* 非自己持有的对象无法释放
* autorelease 对象

| 对象操作        | Objective-C 方法 |
|:-------------|:-------------|
| 生成并持有对象     | alloc/new/copy/mutableCopy等方法 |
| 持有对象      |   retain      | 
| 释放对象 | release   | 
| 废弃对象 | dealloc   | 

### 自己生成的对象，自己所持有
下列名称开头的方法名意味着自己生成的对象自己所持有。
* alloc
* new
* copy
* mutableCopy

实例代码
```
id obj = [NSObject new];
```

### 非自己生成的对象，自己也可以持有

用 alloc/new/copy/mutableCopy 以外的方法取得的对象，因为非自己生成并持有，所以不是该对象的持有者。但是我们可以持有它

实例代码：
```
// 取得非自己生成的对象
id obj = [NSMutableArray array];

// 持有非自己生成的对象
[obj retain];
```

### 不再需要自己持有的对象时释放
自己持有的对象，一旦不在需要，持有者有义务释放该对象，释放使用 release。
实例代码：
```
// 自己生成并持有对象
id obj = [[NSObject alloc] init];

// 释放对象
[obj release];
```

同理，用 alloc 方法由自己生成并持有的对象通过 release 方法就释放了。非自己生成而持有的对象，若用 retain 方法变为自己持有，也可以使用 release 方法释放。
```
// 取得非自己生成的对象
id obj = [NSMutableArray array];

// 持有对象
[obj retain];

// 释放对象
[obj release];
```

### 非自己持有的对象无法释放

释放非自己持有的对象会造成崩溃
实例代码:
```
// 自己生成并持有
id obj = [NSObject new];
// 释放持有对象
[obj release];
// 释放非自己持有的对象，因为之前已经释放了。crash ！！！
[obj release];
```

### autorelease 对象
autorelease 对象可以达到对象存在但是自己又不持有的效果。如 `[NSMutableArray array]`

实现方案:
```
- (NSMutableArray *) array
{
      // 创建并持有对象
      id obj = [NSObject new];
      // 取得对象存在，但是自己不持有
     [obj autorelease];
     return obj;
}
```

autorelease 对象会被 autoreleasepool 所持有，在 runloop 休眠的时候会释放所持有的对象。

## GUNstep 的计数实现

将引用计数存在对象占用的内存块头部的变量中。
* 在 Objective-C 的对象中存在引用计数这一个整数值。
* 调用 alloc 或是 retain 方法后，引用计数值加 1。
* 调用 release 后，引用计数减 1。
* 引用计数值为 0 时，调用 dealloc 方法废弃对象。

## 苹果的实现
苹果使用了散列表来管理引用计数。
对比：

GUNstep
* 少量代码即可完成。
* 能够统一管理引用计数用内存块与对象用内存快。

Apple
* 对象用内存快的分配无需考虑内存块头部。
* 引用计数表各记录中存在内存块地址，可以从各个记录追溯到各对象内存块。利于调试。

## autorelease
autorelease 对象可以达到对象存在但是自己又不持有的效果。autorelease 对象离开作用域都将调用 release 实例方法。
具体使用(MRC)：
1. 生成并持有 NSAutoreleasePool 对象。
2. 调用已分配对象的 autorelease 实例方法。
3. 废弃 NSAutoreleasePool 对象。


实例代码：
```
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
id obj = [NSObject new];
[obj autorelease];
[pool drain];
```

## ARC 规则

所有权修饰符
* __strong 修饰符
* __weak 修饰符
* __unsafe_unretained 修饰符
* __autorelease 修饰符

### __strong 修饰符
__strong 是默认修饰符
__strong 修饰符表示对对象的强引用。持有强引用的变量在超出其作用域时被废弃，随着强引用失效，引用的对象会随之释放。
* 如果是自己生成的对象正常持有。
* 如果不是自己生成的对象，会自动加 retain 来持有。

### __weak 修饰符
__strong 修饰符会造成循环引用问题，__weak 修饰符可以避免循环引用。__weak 不持有对象，在对象被释放的时候会自动设置为 nil。
__weak 修饰符只能用于 iOS 5 以上以及 OS X Lion 以上版本，其他版本需要使用 __unsafe_unretained 来替代。

### __unsafe_unretained 修饰符
__unsafe_unretained 修饰符是不安全的所有权修饰符。尽管 ARC 是由编译器管理的，但是  __unsafe_unretained 修饰符修饰的变量不属于编译器的内存管理对象。
和 __weak 不持有对象一样。释放了不会被设置为 nil，会出现野指针的情况。

### __autorelease 修饰符
在 ARC 下不能显示的调用 autorelease 方法，那么需要使用 __autorelease 修饰。
实例代码:
```
// MRC 
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
id obj = [NSObject new];
[obj autorelease];
[pool drain];

// ARC
@autoreleasepool{
  id __autorelease obj = [[NSObject alloc] init];
}
```

知识点:
1. 访问 weak 变量是必须访问注册到 autorelease 的对象，因为 weak 修饰符只持有对象的弱引用，而在访问过程中，该对象可能被废弃。如果把要访问的对象注册到 autoreleasepool 中，那么在 runloop 周期内都不会给释放。
2. id 的指针和对象的指针（如 `NSError ** `）没有显示的修饰符那么默认是 __autorelease.

## ARC 下编写代码规则
* 不能使用 retain / release / retainCount /autorelease。
* 不能使用 NSAllocateObject / NSDeallocateObject。
* 必须遵守内存管理的方法命名规则。
* 不要显式的调用 delloc。
* 使用 @autoreleasepool 块来替代 NSAutoreleasePool。
* 不能使用区域（NSZone）
* 对象型变量不能作为 C 语言结构体的成员。
* 显示转换 id 和 void *。

## Toll-Free Bridge
Core Foundation 对象和 Objective-C 对象没有区别，不同之处只是由哪个框架生成对象， Core Foundation 对象可以转换成 Objective-C 对象来使用，转换过程称之为 `Toll-Free Bridge`。有以下几个关键字
* __bridge ：**只做类型转换，但是不修改对象（内存）管理权**。
* __bridge_retained：**将Objective-C的对象转换为Core Foundation的对象**，同时将对象（内存）的管理权交给我们，后续需要使用CFRelease或者相关方法来释放对象。
* __bridge_transfer：**将Core Foundation的对象转换为Objective-C的对象**，同时将对象（内存）的管理权交给ARC。

## ARC 实现
### strong 修饰符
编译器来管理，2 种情况

1. alloc/new/copy/mutableCopy等方法生成的对象，会自动插入 `release `方法。
2. 非情况 1 方法生成的对象，如 `[NSMutableArray array]` ，会分别插入`objc_autoreleaseReturnValue `和 `objc_retainAutoreleasedReturnValue`方法。

`objc_autoreleaseReturnValue `和`objc_retainAutoreleasedReturnValue `是成对出现的。在对象返回处调用`objc_autoreleaseReturnValue `，并且强引用的时候紧接着会调用`objc_retainAutoreleasedReturnValue `。
系统会对这个其中赋值过程进行优化。

![过程图解.png](/img/2017-1-8-object-c-advanced.png)

### weak 修饰符
weak 对象流程
1. 初始化 weak：`objc_initWeak`
2. 改变 weak 值：`objc_storeWeak `
3. 释放 weak 值：`objc_destroyWeak`

# Blocks
## 什么是 Blocks
Blocks 是 C 语言的扩充功能：带有局部变量的匿名函数。
其他语言中的 Blocks 的名称

| 语言        | 名称 |
|:-------------|:-------------|
| C     | Blocks |
| Smalltalk      |   Blocks      | 
| Ruby | Blocks   | 
| LISP | Lambda   | 
| Python | Lambda   | 
| C++ | Lambda   | 

语法
```
^ 返回值类型 参数列表{
  表达式
}
```

## Blocks 变量
语法（用在属性，变量处）
```
返回值类型 (^ 变量名)参数列表
```

## 截获自动变量
* 普通变量获取值得内容
* 指针变量强引用指针对象

## __block
截获的自动变量不能子啊 Blocks 块内被修改，需要加 __block 修饰符。

## Blocks 的实现
一个 Blocks 被编译后会生成这么几个结构体
* __block_impx:包含了 isa，标志位，保留字段，执行的函数指针。
* __xxx_block_desc_x:包含了保留字段，block 的大小。
* `xxx_block_impx_xxx`：包含了上面 2 个成员。

x 表示由编译器决定的名字。

### 截获自动变量的原理
截获的变量，会在` __block_impx` 中生成相应的成员变量。普通变量值是拷贝值，指针则是强引用指针。

### __block 修饰符
当用 __block 修饰符的时候，会多生成`__Block_byref_val_x`，包含了被捕获的值，isa，forwarding 指针等字段。
之所以可以在 block 内部修改 __block 变量因为生成这个结构体来包装了一层，通过这个结构体达到修改外部变量的目的。

### forwarding 指针
当一个栈 block 执行出了作用域的时候，其捕获的 __block 变量也会出作用域，会出现野指针，如果把 __block 拷贝到堆上，这个时候这个 __block 变量也应该一同拷贝，需要有一个机制在被拷贝之后也会引用到系统的堆变量上。forwarding 指针就是做这个的，forwarding 在未被拷贝的时候始终指向自己，在拷贝之后 forwarding 指针会指向堆变量。防止了野指针的问题，也解决了拷贝之后的引用问题。

## 循环引用
在使用 block 由于捕获变量都是强引用的，所以需要注意循环引用。可以使用 weak 变量来避免循环引用。

更详细的内容可以看我另外一篇[Block 博客](http://www.jianshu.com/p/cdf46bbbfdfe)

未完待续...


