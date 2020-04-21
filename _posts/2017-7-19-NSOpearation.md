---
layout: post
title: "iOS多线程之NSOpearation"
author: "Damien"
# catalog: true
tags:
    - iOS
--- 

在 iOS 开发中，异步操作通常使用 GCD 的方式来完成，GCD 可以简单快速完成异步操作，当如果涉及到高级的操作的时候如异步依赖、暂停、结束、优先级往往会要写很多代码，而 GCD C语言式的写法，看起来也不是那么的“优雅”，NSOpearation 是基于 GCD 的封装，提供面向对象的形式，容易将异步任务封装成一个个异步“操作”对象，也会让代码组织更加优雅，许多第三方库都在使用 NSOpearation 来完成异步任务，如 SDWebimage，NSOpearation 在 GCD 之外提供我们另外一个选择。

# NSOpearation
`NSOpearation `表示了一个独立的异步操作单位，它是一个抽象类，我们不能直接使用它，我们需要使用它的子类

* NSInvocationOperation
* NSBlockOperation
* 自定义的 NSOpearation

其中 NSInvocationOperation 和 NSBlockOperation 是系统提供，如果不满足我们可以使用自定义。自定义我们在后文讲。


## NSInvocationOperation 与 NSBlockOperation

### 创建 Opearation
`NSInvocationOperation `是通过一个 object 和 selector 来生成一个 Operation， 如下代码

```
   NSInvocationOperation *invocationOperation = [[NSInvocationOperation alloc]initWithTarget:self selector:@selector(doSomething) object:nil];
 [invocationOperation start];
```

`NSBlockOperation `可以通过一个或者多个 block 来生成 Operation

```
    NSBlockOperation *blockOperation = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"block2");
    }];
    
    [blockOperation addExecutionBlock:^{
        NSLog(@"block2");
    }];
    
    [blockOperation start];
```
### 执行Opearation

NSOpearation 及其子类有 2 种执行方式
1. 调用 start
2. 添加到 NSOperationQueue

start 方法：start 方法一般会在调用 start 方法的线程执行，会阻塞当前线程，**但 NSBlockOperation 中如果添加了多个执行的 block 时，不一定会在调用的线程中执行**。

添加到 NSOperationQueue：当一个 NSOpearation 被添加到 NSOperationQueue 中的时候，NSOperationQueue 会为 NSOpearation 提供线程，这个时候始终是在异步线程的中执行。

### 同步的 Opearation 和异步 Opearation
当我们使用 start 执行一个 Opearation 时，这个 Opearation 将会在调用 start 方法的线程执行，这是同步的。会阻塞的线程，这叫做同步的 Opearation，当然也可以在 start 方法中配置的执行的线程环境，使达到在异步执行的效果，或者将 Opearation 加入 NSOperationQueue 中执行，那么 NSOperationQueue 会自动给你切换到异步线程，这就叫异步 Opearation。

2 种形式的代码如下：

```
    NSBlockOperation *blockOperation = [NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"block");
    }];
//    [blockOperation start];
    NSOperationQueue *queue = [NSOperationQueue new];
    [queue addOperation:blockOperation];
```

## 自定义Operation
如果 NSInvocationOperation 与 NSBlockOperation 不能很好完成我们的需求，我们可以选择自定义 Operation，继承 NSOperation 即可实现自己的 Operation，当然也需要多花点代码来维护自己的 Operation

### 需要维护状态
作为自定义的 Operation 我们需要一些状态来维护外界对我们的观察。

#### asynchronous（非必要）
这个属性表示当前 Operation 时候需要异步运行，默认是 NO。这个属性对 Operation 执行线程没有关系，即使返回 YES，如果是直接手动调用 start 方法，那么这个 Operation 也是运行在调用线程中。

#### cancelled
 当 Operation 的 cancel 方法被调用的时候， cancelled 会被设置为 YES，Operation 执行一个长时间的异步任务的时候，需要经常检查这个 Operation 是否被取消，也就是 cancelled 是否为 YES，如果检测到 Operation 被取消，我们需要及时响应取消任务事件。

#### executing （必须维护）
表明当前的 Operation 正在执行中，需要自己维护

#### finished （必须维护）
表明当前的 Operation 正在已经结束，需要自己维护，如果对 Operation 添加依赖，那么这个属性很重要。因为依赖的执行取决于被依赖的那个 Operation 的 finished 是不是 YES， 否则不会开始依赖任务。

###需要重写的方法
#### start
如果你的 Operation 不是异步执行的，那么这个方法你可以不用重写，如果你的 Operation 是一个异步执行的任务，那么需要重写 start 方法（不用调 super 的 start）来为 Operation 配置线程和一些初始化操作，当然你的 Operation 是加到 NSOperationQueue 中的，那么你也可以选择不重写这个 start 方法。

#### main
重写 main 方法（不用调用super 的 main）是可选择的，尽管在 start 方法我们可以写执行任务的代码，但是我们可以把执行任务的代码放到 main 中达到配置和执行分离的目的，使得代码结构变得清晰，这里有个地方要注意，如果你重写了 start 方法，那么记得调用下 main 方法，因为默认的 start 方法是会调用的，重写了需要我们手动加上。

### 维护 KVO 通知

在我们重写了大量方法后，我们需要自己维护 KVO 通知，如以下几个通知我们应该按需来维护

isCancelled
isConcurrent
isExecuting
isFinished
isReady
dependencies
queuePriority
completionBlock

## Operation 的依赖
我们可以使用

```
- (void)addDependency:(NSOperation *)op;
- (void)removeDependency:(NSOperation *)op;
```
这 2 个方法来为 Operation 添加依赖，添加依赖之后，**被依赖的那个 Operation 结束后，并且 isFinshed 状态为 YES** ，这个时候依赖的 Operation 才会运行，**添加依赖关系后，即使加入不同的 OperationQueue 它们的依赖关系一样存在。**

## Operation 的优先级
除了设置依赖，我们还可以设置 Operation 的优先级，优先级高的会被优先级低的优先执行。系统提供了 5 个等级
```
typedef NS_ENUM(NSInteger, NSOperationQueuePriority) {
	NSOperationQueuePriorityVeryLow = -8L,
	NSOperationQueuePriorityLow = -4L,
	NSOperationQueuePriorityNormal = 0,
	NSOperationQueuePriorityHigh = 4,
	NSOperationQueuePriorityVeryHigh = 8
};
```
**优先级生效的必要条件是 Operation 的 isReady 为 YES 状态**。
比如，在一个 operation queue 中，有一个高优先级和一个低优先级的 operation ，并且它们的 isReady 状态都为 YES ，那么高优先级的 operation 将会优先执行。而如果这个高优先级的 operation 的 isReady 状态为 NO ，而低优先级的 operation 的 isReady 状态为 YES 的话，那么这个低优先级的 operation 反而会优先执行。
##  Completion Block
在 Operation 结束后，我们可以为它设置一个 Completion Block 来获取完成之后的操作。我们没有办法保证 completion block 被回调时一定是在主线程，理论上它应该是与触发 isFinished 的 KVO 通知所在的线程一致的。

# Operation Queue
Operation Queue 是 Operation 的载体，可以为 Operation 提供线程，提供依赖关系，提供优先级机制和线程调度。我们可以把 Operation 加入到 Queue 中运行。

## 添加 Operation
 我们可以使用下面 3 个 API 来把 Operation 添加到 Queue，
  
```
addOperation: ，添加一个 operation 到 operation queue 中；
addOperations:waitUntilFinished: ，添加一组 operation 到 operation queue 中；
addOperationWithBlock: ，直接添加一个 block 到 operation queue 中，而不用创建一个 NSBlockOperation 对象。
```
## 控制最大的并发数量
设置 setMaxConcurrentoperationCount 可以控制并发数量，setMaxConcurrentoperationCount 为 1 也可以当做是一个串行队列。

## 暂停和恢复
通过设置 suspended 属性，可以挂起，和继续队列，也就是暂停和恢复，暂停执行 operation queue 并不能使正在执行的 operation 暂停执行，而只是简单地暂停调度新的 operation 。