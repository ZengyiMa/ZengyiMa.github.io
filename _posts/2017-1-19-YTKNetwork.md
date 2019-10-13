---
layout: post
title: "YTKNetwork源码阅读"
author: "Damien"
# catalog: true
tags:
    - iOS
    - 源码阅读
--- 


## 关于YTKNetwork

YTKNetwork是猿题库开发的一套基于AFNetworking的网络请求库，提供将AF请求封装成一个个接口对象来进行网络请求，提供更高级的抽象。

> YTKNetwork 是猿题库 iOS 研发团队基于 AFNetworking 封装的 iOS 网络库，其实现了一套 High Level 的 API，提供了更高层次的网络访问抽象。YTKNetwork 现在同时被使用在猿题库公司的所有产品的 iOS 端，包括：猿题库、 小猿搜题、 猿辅导 、 粉笔直播课 。

## YTKNetwork提供的功能

- 支持按时间缓存网络请求内容


- 支持按版本号缓存网络请求内容


- 支持统一设置服务器和 CDN 的地址


- 支持检查返回 JSON 内容的合法性


- 支持文件的断点续传


- 支持 block 和 delegate 两种模式的回调方式


- 支持批量的网络请求发送，并统一设置它们的回调（实现在YTKBatchRequest类中）


- 支持方便地设置有相互依赖的网络请求的发送，例如：发送请求A，根据请求A的结果，选择性的发送请求B和C，再根据B和C的结果，选择性的发送请求D。（实现在YTKChainRequest类中）


- 支持网络请求 URL 的 filter，可以统一为网络请求加上一些参数，或者修改一些路径。


- 定义了一套插件机制，可以很方便地为 YTKNetwork 增加功能。猿题库官方现在提供了一个插件，可以在某些网络请求发起时，在界面上显示"正在加载"的 HUD。

## 梳理关键类

`YTKNetworkPrivate` ：包含着一些便利方法，如 JSON 合法性，MD5 编码的方法， 从请求字典解析出参数等方法。

`YTKNetworkConfig`：网络配置方法，如基本 URL ，CDN 地址等

`YTKBaseRequest` ：YTKNetwork每个请求的虚拟基类，不直接使用，只是提供公共接口

`YTKRequest` ：继承于`YTKBaseRequest`实现其方法，并且添加了缓存相关的功能。如缓存验证，缓存保存等等。

`YTKNetworkAgent`：基本请求的操作代理，所有请求会被放在这里来执行。

## 基本请求

如Demo演示的，一个请求的过程是这样的

```
 GetUserInfoApi *api = [[GetUserInfoApi alloc] initWithUserId:userId];
    if ([api cacheJson]) {
        NSDictionary *json = [api cacheJson];
        NSLog(@"json = %@", json);
        // show cached data
    }
    [api startWithCompletionBlockWithSuccess:^(YTKBaseRequest *request) {
        NSLog(@"update ui");
    } failure:^(YTKBaseRequest *request) {
        NSLog(@"failed");
    }];
```

我们的请求，需要继承`YTKRequest`并实现`requestUrl`，`requestArgument`等关键方法。

然后调用`startWithCompletionBlockWithSuccess`来开始一个请求。

在`startWithCompletionBlockWithSuccess`做了什么事呢？

在此方法中设置了成功和失败的2个Block，并且调用`start`方法,在`start`中进行缓存的判断。

```
if (self.ignoreCache) {
        [super start];
        return;
    }

    // check cache time
    if ([self cacheTimeInSeconds] < 0) {
        [super start];
        return;
    }

    // check cache version
    long long cacheVersionFileContent = [self cacheVersionFileContent];
    if (cacheVersionFileContent != [self cacheVersion]) {
        [super start];
        return;
    }

    // check cache existance
    NSString *path = [self cacheFilePath];
    NSFileManager *fileManager = [NSFileManager defaultManager];
    if (![fileManager fileExistsAtPath:path isDirectory:nil]) {
        [super start];
        return;
    }

    // check cache time
    int seconds = [self cacheFileDuration:path];
    if (seconds < 0 || seconds > [self cacheTimeInSeconds]) {
        [super start];
        return;
    }

    // load cache
    _cacheJson = [NSKeyedUnarchiver unarchiveObjectWithFile:path];
    if (_cacheJson == nil) {
        [super start];
        return;
    }

```



当缓存不存在，或者不需要缓存的时候会直接开始请求，

```
- (void)start {
    [self toggleAccessoriesWillStartCallBack];
    [[YTKNetworkAgent sharedInstance] addRequest:self];
}
```

可以看到`YTKNetworkAgent`来进行请求的持有和发送请求。
让我们聚焦`YTKNetworkAgent`内部实现细节。
在`agent`内部，持有了2个，存放请求的数组和`AFHTTPRequestOperationManager`
```
    AFHTTPRequestOperationManager *_manager;
    YTKNetworkConfig *_config;
    NSMutableDictionary *_requestsRecord;
```
在`addRequest`的操作是

1. `buildRequestUrl`:构建请求的URL，这将在你实现的接口类里面获取请求URL和BaseURL相链接，并且使用`YTKUrlFilterProtocol`协议实现一些接口过滤的操作。
2. 获取请求参数，拼接http请求头，
3. 调用AF的get，post，patch等方法
4. 添加到request到record数组

```
- (void)addRequest:(YTKBaseRequest *)request {
    YTKRequestMethod method = [request requestMethod];
    NSString *url = [self buildRequestUrl:request];
    id param = request.requestArgument;
    AFConstructingBlock constructingBlock = [request constructingBodyBlock];

    if (request.requestSerializerType == YTKRequestSerializerTypeHTTP) {
        _manager.requestSerializer = [AFHTTPRequestSerializer serializer];
    } else if (request.requestSerializerType == YTKRequestSerializerTypeJSON) {
        _manager.requestSerializer = [AFJSONRequestSerializer serializer];
    }
    
    _manager.requestSerializer.timeoutInterval = [request requestTimeoutInterval];

    // if api need server username and password
    NSArray *authorizationHeaderFieldArray = [request requestAuthorizationHeaderFieldArray];
    if (authorizationHeaderFieldArray != nil) {
        [_manager.requestSerializer setAuthorizationHeaderFieldWithUsername:(NSString *)authorizationHeaderFieldArray.firstObject
                                                                   password:(NSString *)authorizationHeaderFieldArray.lastObject];
    }
    
    // if api need add custom value to HTTPHeaderField
    NSDictionary *headerFieldValueDictionary = [request requestHeaderFieldValueDictionary];
    if (headerFieldValueDictionary != nil) {
        for (id httpHeaderField in headerFieldValueDictionary.allKeys) {
            id value = headerFieldValueDictionary[httpHeaderField];
            if ([httpHeaderField isKindOfClass:[NSString class]] && [value isKindOfClass:[NSString class]]) {
                [_manager.requestSerializer setValue:(NSString *)value forHTTPHeaderField:(NSString *)httpHeaderField];
            } else {
                YTKLog(@"Error, class of key/value in headerFieldValueDictionary should be NSString.");
            }
        }
    }

    // if api build custom url request
    NSURLRequest *customUrlRequest= [request buildCustomUrlRequest];
    if (customUrlRequest) {
        AFHTTPRequestOperation *operation = [[AFHTTPRequestOperation alloc] initWithRequest:customUrlRequest];
        [operation setCompletionBlockWithSuccess:^(AFHTTPRequestOperation *operation, id responseObject) {
            [self handleRequestResult:operation];
        } failure:^(AFHTTPRequestOperation *operation, NSError *error) {
            [self handleRequestResult:operation];
        }];
        request.requestOperation = operation;
        operation.responseSerializer = _manager.responseSerializer;
        [_manager.operationQueue addOperation:operation];
    } else {
        if (method == YTKRequestMethodGet) {
            if (request.resumableDownloadPath) {
                // add parameters to URL;
                NSString *filteredUrl = [YTKNetworkPrivate urlStringWithOriginUrlString:url appendParameters:param];

                NSURLRequest *requestUrl = [NSURLRequest requestWithURL:[NSURL URLWithString:filteredUrl]];
                AFDownloadRequestOperation *operation = [[AFDownloadRequestOperation alloc] initWithRequest:requestUrl
                                                                                                 targetPath:request.resumableDownloadPath shouldResume:YES];
                [operation setProgressiveDownloadProgressBlock:request.resumableDownloadProgressBlock];
                [operation setCompletionBlockWithSuccess:^(AFHTTPRequestOperation *operation, id responseObject) {
                    [self handleRequestResult:operation];
                }                                failure:^(AFHTTPRequestOperation *operation, NSError *error) {
                    [self handleRequestResult:operation];
                }];
                request.requestOperation = operation;
                [_manager.operationQueue addOperation:operation];
            } else {
                request.requestOperation = [_manager GET:url parameters:param success:^(AFHTTPRequestOperation *operation, id responseObject) {
                    [self handleRequestResult:operation];
                }                                failure:^(AFHTTPRequestOperation *operation, NSError *error) {
                    [self handleRequestResult:operation];
                }];
            }
        } else if (method == YTKRequestMethodPost) {
            if (constructingBlock != nil) {
                request.requestOperation = [_manager POST:url parameters:param constructingBodyWithBlock:constructingBlock
                                                  success:^(AFHTTPRequestOperation *operation, id responseObject) {
                                                      [self handleRequestResult:operation];
                                                  } failure:^(AFHTTPRequestOperation *operation, NSError *error) {
                            [self handleRequestResult:operation];
                        }];
            } else {
                request.requestOperation = [_manager POST:url parameters:param success:^(AFHTTPRequestOperation *operation, id responseObject) {
                    [self handleRequestResult:operation];
                }                                 failure:^(AFHTTPRequestOperation *operation, NSError *error) {
                    [self handleRequestResult:operation];
                }];
            }
        } else if (method == YTKRequestMethodHead) {
            request.requestOperation = [_manager HEAD:url parameters:param success:^(AFHTTPRequestOperation *operation) {
                [self handleRequestResult:operation];
            }                                 failure:^(AFHTTPRequestOperation *operation, NSError *error) {
                [self handleRequestResult:operation];
            }];
        } else if (method == YTKRequestMethodPut) {
            request.requestOperation = [_manager PUT:url parameters:param success:^(AFHTTPRequestOperation *operation, id responseObject) {
                [self handleRequestResult:operation];
            }                                failure:^(AFHTTPRequestOperation *operation, NSError *error) {
                [self handleRequestResult:operation];
            }];
        } else if (method == YTKRequestMethodDelete) {
            request.requestOperation = [_manager DELETE:url parameters:param success:^(AFHTTPRequestOperation *operation, id responseObject) {
                [self handleRequestResult:operation];
            }                                   failure:^(AFHTTPRequestOperation *operation, NSError *error) {
                [self handleRequestResult:operation];
            }];
        } else if (method == YTKRequestMethodPatch) {
            request.requestOperation = [_manager PATCH:url parameters:param success:^(AFHTTPRequestOperation *operation, id responseObject) {
                [self handleRequestResult:operation];
            } failure:^(AFHTTPRequestOperation *operation, NSError *error) {
                [self handleRequestResult:operation];
            }];
        } else {
            YTKLog(@"Error, unsupport method type");
            return;
        }
    }

    YTKLog(@"Add request: %@", NSStringFromClass([request class]));
    [self addOperation:request];
}

```

之后就是漫长的网络请求。请求完成又是怎么处理的呢？

**请求回调的处理**在方法`handleRequestResult`中。其原理是判断是否请求成功，如验证服务器返回是不是成功，失败，或者网络连接失败，之后是调用delegate的回调，在往下是完成block。最后将request移除数组。
```
- (void)handleRequestResult:(AFHTTPRequestOperation *)operation {
    NSString *key = [self requestHashKey:operation];
    YTKBaseRequest *request = _requestsRecord[key];
    YTKLog(@"Finished Request: %@", NSStringFromClass([request class]));
    if (request) {
        BOOL succeed = [self checkResult:request];
        if (succeed) {
            [request toggleAccessoriesWillStopCallBack];
            [request requestCompleteFilter];
            if (request.delegate != nil) {
                [request.delegate requestFinished:request];
            }
            if (request.successCompletionBlock) {
                request.successCompletionBlock(request);
            }
            [request toggleAccessoriesDidStopCallBack];
        } else {
            YTKLog(@"Request %@ failed, status code = %ld",
                     NSStringFromClass([request class]), (long)request.responseStatusCode);
            [request toggleAccessoriesWillStopCallBack];
            [request requestFailedFilter];
            if (request.delegate != nil) {
                [request.delegate requestFailed:request];
            }
            if (request.failureCompletionBlock) {
                request.failureCompletionBlock(request);
            }
            [request toggleAccessoriesDidStopCallBack];
        }
    }
    [self removeOperation:operation];
    [request clearCompletionBlock];
}

```



