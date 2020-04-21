---
layout: post
title: "AFNetworking 核心解析"
author: "Damien"
# catalog: true
tags:
    - iOS
    - 源码阅读
--- 


# AFNetworking 核心解析

AFNetworing 是 iOS 最常用的网络请求库，在最新的版本已经使用了 NSURLSession 来实现。

# 概况
AF 中有这么几个类

`AFHTTPSessionManager`：最常用的对外的接口，提供 HTTP 请求的接口，如 get，post，head 等。
`AFNetworkReachabilityManager`：网络监控类。
`AFSecurityPolicy`：请求安全策略类。
`AFURLRequestSerialization`：请求参数序列化类
`AFURLResponseSerialization`：请求返回数据解析类
`AFURLSessionManager`：对 NSURLSeesion 的封装，核心请求类。
`AFURLSessionManagerTaskDelegate`：NSURLSession 回调的封装。内部类。

本文为核心解析，丢弃一些细节，直入主题，`AFHTTPSessionManager`继承于`AFURLSessionManager`，实际请求使用`AFURLSessionManager`提供的接口来完成。所以本文直接介绍`AFURLSessionManager`。其他部分将会拆分其他的文章。

# AFURLSessionManager
`AFURLSessionManager`是最核心的请求类。提供了对 NSURLSession 的封装。
## 初始化
`AFURLSessionManager`提供了一个参数为`NSURLSessionConfiguration`的指定构造器。如果没给`NSURLSessionConfiguration`那么默认将会是`defaultSessionConfiguration`

```
- (instancetype)initWithSessionConfiguration:(NSURLSessionConfiguration *)configuration {
    self = [super init];
    if (!self) {
        return nil;
    }

    if (!configuration) {
        configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    }

    self.sessionConfiguration = configuration;

    self.operationQueue = [[NSOperationQueue alloc] init];
    self.operationQueue.maxConcurrentOperationCount = 1;

    self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration delegate:self delegateQueue:self.operationQueue];

    self.responseSerializer = [AFJSONResponseSerializer serializer];

    self.securityPolicy = [AFSecurityPolicy defaultPolicy];

#if !TARGET_OS_WATCH
    self.reachabilityManager = [AFNetworkReachabilityManager sharedManager];
#endif

    self.mutableTaskDelegatesKeyedByTaskIdentifier = [[NSMutableDictionary alloc] init];

    self.lock = [[NSLock alloc] init];
    self.lock.name = AFURLSessionManagerLockName;

    [self.session getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) {
        for (NSURLSessionDataTask *task in dataTasks) {
            [self addDelegateForDataTask:task uploadProgress:nil downloadProgress:nil completionHandler:nil];
        }

        for (NSURLSessionUploadTask *uploadTask in uploadTasks) {
            [self addDelegateForUploadTask:uploadTask progress:nil completionHandler:nil];
        }

        for (NSURLSessionDownloadTask *downloadTask in downloadTasks) {
            [self addDelegateForDownloadTask:downloadTask progress:nil destination:nil completionHandler:nil];
        }
    }];

    return self;
}

```

由此也可以看出，线程安全使用的锁为`NSLock`

## 请求
NSURLSession 分成 3 种请求，分别为 dataTask，uploadTask 和 downloadTask。在`AFURLSessionManager`也暴露出了这 3 种请求。

dataTask

```
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                               uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                             downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                            completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject,  NSError * _Nullable error))completionHandler {

    __block NSURLSessionDataTask *dataTask = nil;
    url_session_manager_create_task_safely(^{
        dataTask = [self.session dataTaskWithRequest:request];
    });
    
    [self addDelegateForDataTask:dataTask uploadProgress:uploadProgressBlock downloadProgress:downloadProgressBlock completionHandler:completionHandler];

    return dataTask;
}
```

uploadTask

```
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request
                                         fromData:(NSData *)bodyData
                                         progress:(void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                                completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    __block NSURLSessionUploadTask *uploadTask = nil;
    url_session_manager_create_task_safely(^{
        uploadTask = [self.session uploadTaskWithRequest:request fromData:bodyData];
    });

    [self addDelegateForUploadTask:uploadTask progress:uploadProgressBlock completionHandler:completionHandler];

    return uploadTask;
}
```

downloadTask

```
- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request
                                             progress:(void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                                          destination:(NSURL * (^)(NSURL *targetPath, NSURLResponse *response))destination
                                    completionHandler:(void (^)(NSURLResponse *response, NSURL *filePath, NSError *error))completionHandler
{
    __block NSURLSessionDownloadTask *downloadTask = nil;
    url_session_manager_create_task_safely(^{
        downloadTask = [self.session downloadTaskWithRequest:request];
    });

    [self addDelegateForDownloadTask:downloadTask progress:downloadProgressBlock destination:destination completionHandler:completionHandler];

    return downloadTask;
}
```

这 3 个方法流程都很相似，使用 NSURLSession 来获取 task，之后将 3 种 task 添加到相应的 delegate 中，如 dataTask


```
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] init];
    delegate.manager = self;
    delegate.completionHandler = completionHandler;

    dataTask.taskDescription = self.taskDescriptionForSessionTasks;
    [self setDelegate:delegate forTask:dataTask];

    delegate.uploadProgressBlock = uploadProgressBlock;
    delegate.downloadProgressBlock = downloadProgressBlock;
```
添加 delegate 的方法大概分成几部，首先生成`AFURLSessionManagerTaskDelegate`对象，此对象对应每次请求 task 的回调处理，将 completionHandler 请求完成的回调设置到 AFURLSessionManagerTaskDelegate 对象中，之后调用

```
    [self setDelegate:delegate forTask:dataTask];

```
来把 task 添加到以 task 的 taskIdentifier 为 key，value 为 task 的字典中，后文中也会通过此字典来获取每一个 task

```
- (void)setDelegate:(AFURLSessionManagerTaskDelegate *)delegate
            forTask:(NSURLSessionTask *)task
{
    NSParameterAssert(task);
    NSParameterAssert(delegate);

    [self.lock lock];
    self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)] = delegate;
    [delegate setupProgressForTask:task];
    [self addNotificationObserverForTask:task];
    [self.lock unlock];
}
```

# 请求回调
`AFURLSessionManager`实现了 NSURLSeesion 的几个回调，除了处理 HTTPS 之外的所有回调都委托到各个的 task 的 AFURLSessionManagerTaskDelegate 来处理。


```
- (void)URLSession:(NSURLSession *)session
          dataTask:(NSURLSessionDataTask *)dataTask
    didReceiveData:(NSData *)data
{

    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:dataTask];
    [delegate URLSession:session dataTask:dataTask didReceiveData:data];

    if (self.dataTaskDidReceiveData) {
        self.dataTaskDidReceiveData(session, dataTask, data);
    }
}
```
通过

```
    AFURLSessionManagerTaskDelegate *delegate = [self delegateForTask:dataTask];
```
来获取 task 的 AFURLSessionManagerTaskDelegate。

```
- (AFURLSessionManagerTaskDelegate *)delegateForTask:(NSURLSessionTask *)task {
    NSParameterAssert(task);

    AFURLSessionManagerTaskDelegate *delegate = nil;
    [self.lock lock];
    delegate = self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)];
    [self.lock unlock];

    return delegate;
}
```
通过 taskIdentifier 在字典中取得 AFURLSessionManagerTaskDelegate。

# AFURLSessionManagerTaskDelegate
此类的主要作用是对每个 task 的 delegate 进行封装。封装独立性，包括下载的 task 的 downloadURL 处理等。


```
- (void)URLSession:(__unused NSURLSession *)session
              task:(NSURLSessionTask *)task
didCompleteWithError:(NSError *)error
{
    __strong AFURLSessionManager *manager = self.manager;

    __block id responseObject = nil;

    __block NSMutableDictionary *userInfo = [NSMutableDictionary dictionary];
    userInfo[AFNetworkingTaskDidCompleteResponseSerializerKey] = manager.responseSerializer;

    //Performance Improvement from #2672
    NSData *data = nil;
    if (self.mutableData) {
        data = [self.mutableData copy];
        //We no longer need the reference, so nil it out to gain back some memory.
        self.mutableData = nil;
    }

    if (self.downloadFileURL) {
        userInfo[AFNetworkingTaskDidCompleteAssetPathKey] = self.downloadFileURL;
    } else if (data) {
        userInfo[AFNetworkingTaskDidCompleteResponseDataKey] = data;
    }

    if (error) {
        userInfo[AFNetworkingTaskDidCompleteErrorKey] = error;

        dispatch_group_async(manager.completionGroup ?: url_session_manager_completion_group(), manager.completionQueue ?: dispatch_get_main_queue(), ^{
            if (self.completionHandler) {
                self.completionHandler(task.response, responseObject, error);
            }

            dispatch_async(dispatch_get_main_queue(), ^{
                [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingTaskDidCompleteNotification object:task userInfo:userInfo];
            });
        });
    } else {
        dispatch_async(url_session_manager_processing_queue(), ^{
            NSError *serializationError = nil;
            responseObject = [manager.responseSerializer responseObjectForResponse:task.response data:data error:&serializationError];

            if (self.downloadFileURL) {
                responseObject = self.downloadFileURL;
            }

            if (responseObject) {
                userInfo[AFNetworkingTaskDidCompleteSerializedResponseKey] = responseObject;
            }

            if (serializationError) {
                userInfo[AFNetworkingTaskDidCompleteErrorKey] = serializationError;
            }

            dispatch_group_async(manager.completionGroup ?: url_session_manager_completion_group(), manager.completionQueue ?: dispatch_get_main_queue(), ^{
                if (self.completionHandler) {
                    self.completionHandler(task.response, responseObject, serializationError);
                }

                dispatch_async(dispatch_get_main_queue(), ^{
                    [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingTaskDidCompleteNotification object:task userInfo:userInfo];
                });
            });
        });
    }
}

```
处理流程为

1. 判断是否请求成功如果失败，填充`userInfo`，并调用`completionHandler`，传递`response`和`error`更上层。
2. 成功的话，填充`userInfo`，并且调用`completionHandler`，传递 error 为 nil


# 小结
AFNetworking 使用 NSURLSession 来进行网络请求。
AFNetworking 内部使用的锁为 NSLock
AFNetworking 内部维护了一个以 task 的字典，key 为 task 的 taskIdentifier。
AFNetworking 对 每个 task 的 delegate 进行封装。系统只提供对 NSURLSession 的 delegate，对每个 task 不友好。

