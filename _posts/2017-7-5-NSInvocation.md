---
layout: post
title: "NSInvocation"
author: "Damien"
# catalog: true
tags:
    - iOS
    - Objective-C
--- 



在iOS中，可以使用`performSelector`系列方法来对某个类来进行调用方法。

```
- (id)performSelector:(SEL)aSelector;
- (id)performSelector:(SEL)aSelector withObject:(id)object;
- (id)performSelector:(SEL)aSelector withObject:(id)object1 withObject:(id)object2;
```
这种调用有一定局限性，只能最多传递2个参数。一旦参数多了，就无能为力了。但是我们还可以使用`NSInvocation`来进行方法调用。

####NSInvocation
文档描述
>An NSInvocation is an Objective-C message rendered static, that is, it is an action turned into an object. NSInvocation objects are used to store and forward messages between objects and between applications, primarily by NSTimer objects and the distributed objects system.

下面来看一段示例代码

```
 //获取方法签名
  NSMethodSignature *methondSign =  [[self class]instanceMethodSignatureForSelector:@selector(sayHelloTo:)];
    
    //通过方法签名获取调用
   NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:methondSign];
    
    //设置target和selector
    [invocation setTarget:self];
    [invocation setSelector:@selector(sayHelloTo:)];
    
    //设置参数
    NSString *s = @"world";
    [invocation setArgument:&s atIndex:2];
    
    //执行调用
    [invocation invoke];
    
    //如果有延时操作 可以retain参数不被释放
   // [invocation retainArguments];
    
    //如果有返回值  如果是void 会野指针
    char *type = methondSign.methodReturnType;
    if (!strcmp(type, @encode(void))) {
        //没有返回值
    }
    else
    {
        if (!strcmp(type, @encode(id))) {
            //对象类型
            NSString *returnValue = nil;
            [invocation getReturnValue:&returnValue];
            NSLog(@"return value = %@", returnValue);
        }
    }

```

1、获取方法签名
2、通过方法来获取`NSInvocation`对象
3、设置`NSInvocation`的target和selector
4、设置参数，参数从index为2开始，因为前面2个参数被self，_cmd占用。
5、调用`invoke`来执行调用（如果要延时操作可以使用retain来持有不被释放）
6、（如果需要获取返回值）可以先获取返回值type，如果是有类型的可以使用@encode来进行比较，来获取具体的返回值类型，随后调用`getReturnValue`来得到返回值。
