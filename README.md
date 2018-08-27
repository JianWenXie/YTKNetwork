# YTKNetworkAnalysis



## YTKNetwork 源码分析


### 1. 功能介绍   

#### 1.1 YTKNetwork
YTKNetwork 基于 AFNetWorking 封装的一个网络请求库，相比AFNetworking，YTKNetwork 提供了以下更高级的功能:    

 * 支持按时间缓存网络请求内容
 * 支持按版本号缓存网络请求内容
 * 支持统一设置服务器和 CDN 的地址
 * 支持检查返回 JSON 内容的合法性
 * 支持文件的断点续传
 * 支持 block 和 delegate 两种模式的回调方式
 * 支持批量的网络请求发送，并统一设置它们的回调（实现在YTKBatchRequest类中）
 * 支持方便地设置有相互依赖的网络请求的发送，例如：发送请求A，根据请求A的结果，选择性的发送请求B和C，再根据B和C的结果，选择性的发送请求D。（实现在YTKChainRequest类中）
 * 支持网络请求 URL 的 filter，可以统一为网络请求加上一些参数，或者修改一些路径   
 * 定义了一套插件机制，可以很方便地为 YTKNetwork 增加功能。猿题库官方现在提供了一个插件，可以在某些网络请求发起时，在界面上显示“正在加载”的 HUD
    
#### 1.2  YTKNetwork 的基本使用
##### Step 1: 如需统一为网络请求接口加上一些参数和设置统一的服务器，需要在入口类的application didFinishLaunchingWithOptions:中进行如下配置：    

	- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    	NSString *appVersion = [[[NSBundle mainBundle] infoDictionary] objectForKey:@"CFBundleShortVersionString"];
    	//设置统一的服务器接口，可通过如下的baseUrl 进行设置（也可不同过详见下文）如不需要，可忽略
    	YTKNetworkConfig *config = [YTKNetworkConfig sharedConfig];
    	config.baseUrl = @"http://api.alilo.com.cn";
	config.cdnUrl = @"http://fen.bi";
    	// Filter 表示统一为所有的请求添加一个或多个参数(例如统一添加版本号参数)  （如不需要，可忽略）		
    	YTKUrlArgumentsFilter *urlFilter = [YTKUrlArgumentsFilter filterWithArguments:@{@"version": appVersion}];
    	[config addUrlFilter:urlFilter];
    	return YES;
	}


设置好之后，所有的网络请求都会默认使用 YTKNetworkConfig 中 baseUrl 参数指定的地址。
大部分企业应用都需要对一些静态资源（例如图片、js、css）使用 CDN。YTKNetworkConfig 的 cdnUrl 参数用于统一设置这一部分网络请求的地址。
当我们需要切换服务器地址时，只需要修改 YTKNetworkConfig 中的 baseUrl 和 cdnUrl 参数即可。

##### Step 2:构建请求对象继承于父类(YTKRequest)，重写父类的方法,填写参数。
例：修改请求路径需要重写父类的 requestUrl 方法   
	
	- (NSString *)requestUrl {
    		return @"/adidata/hotcity.html";
	}
	以及填写请求参数requestArgument方法
	- (id)requestArgument {
    return @{
             @"app_installtime": @"1433937421",
             @"client_id": @"qyer_ios",
             @"client_secret": @"cd254439208ab658ddf9",
             @"count": @"10",
             @"lat": @"34.72321769638572",
             @"lon": @"113.7348921712425",
             @"page": @"1",
             @"track_app_channel": @"App%2520Store",
             @"track_app_version": @"6.3",
             @"track_device_info": @"iPad%2520mini%28Wifi%29",
             @"track_deviceid": @"C113A3BD-0DD9-4EB1-9BFE-ACFC1590ABBC",
             @"track_os": @"ios%25207.1.2",
             @"type": @"index",
             @"v": @"1"
             };
	}

##### step 3: 开始请求
例:TestGETRequest：YTKRequest    

	TestGETRequest *test = [[TestGETRequest alloc] init];
    [test startWithCompletionBlockWithSuccess:^(__kindof YTKBaseRequest *request) {
        NSLog(@" responseJSONObject ==  \n %@",request.responseJSONObject);
    } failure:^(__kindof YTKBaseRequest *request) {
        NSLog(@"requestOperationError == %@",request.requestOperationError);    
    }];

相关教程和 Demo 可参考

 * [基础教程](https://github.com/yuantiku/YTKNetwork/blob/master/Docs/BasicGuide_cn.md)
 * [高级教程](https://github.com/yuantiku/YTKNetwork/blob/master/Docs/ProGuide_cn.md)
  

### 2. 总体设计(图文)   

#### 2.1总体结构图
![](/pics/flowchart.png )
上面是UIL的总体设计图，整个库分为 YTKRequest，YTKNetworkAgent,YTKNetworkConfig 和 YTKNetworkPrivate 四个模块。    

简而言之，对于我们项目中所有的网络请求结果，当数据从 AFNetWorking 回调回来时，YTKNetworkAgent 先对它进行 JSON 的合法性的校验和版本号缓存，最后再交给最初创建的 YTKRequest 对象进行处理。    
    
#### 2.2UIL 中的概念   

 *  YTKRequest: 负责网络请求根据版本号和时间进行缓存的网络请求基类，继承于 YTKBaseRequest 我们的网络请求如果需要缓存功能都需要继承于YTKRequest；    
 * YTKBatchRequest: 批量网络请求基类；    
 * YTKChainRequest: 链式请求基类；    
 * YTKNetworkAgent: 负责管理所有的网络请求；    
 * YTKNetworkConfig: 负责统一为网络请求加上一些参数，或者修改一些路径；   
 * YTKNetworkPrivate: 负责 JSON 数据合法性的检查,URL 的拼接以及加密等功能。    
  (详见4.2) 

### 3. 流程图
YTKNewWork 网络请求库中最核心的两步：请求过程处理，以及请求到数据时的处理

图3.1    
![](/pics/request_start.png)    
上图为 YTKNetwork 网络开始请求过程流程图：    
下图为 YTKNetWork 处理返回结果流程图：    
图3.2    
![](/pics/new.png)


### 4. 详细设计

#### 4.1类关系UML
![](/pics/iOS网络请求库UML.png)    

#### 4.2  核心类功能介绍

目录：    

[1、YTKNetwork.h](https://github.com/subvin/YTKNetworkAnalysis#4.2.1 YTKBaseRequest)    
[2、YTKBaseRequest](https://github.com/subvin/YTKNetworkAnalysis#4.2.2 YTKBaseRequest )    
[3、YTKRequest](https://github.com/subvin/YTKNetworkAnalysis# 4.2.3 YTKRequest )    
[4、YTKNetworkConfig](https://github.com/subvin/YTKNetworkAnalysis#4.2.4 YTKNetworkConfig )    
[5、YTKNetworkAgent](https://github.com/subvin/YTKNetworkAnalysis# 4.2.5 YTKNetworkAgent )    
[6、YTKBatchRequest](https://github.com/subvin/YTKNetworkAnalysis# 4.2.6 YTKBatchRequest )    
[7、YTKBatchRequestAgent](https://github.com/subvin/YTKNetworkAnalysis#4.2.7 YTKBatchRequestAgent )    
[8、YTKChainRequest](https://github.com/subvin/YTKNetworkAnalysis#4.2.8 YTKChainRequest )    
[9、YTKNetworkPrivate](https://github.com/subvin/YTKNetworkAnalysis#4.2.9 YTKNetworkPrivate )


##### 4.2.1 YTKBaseRequest    

功能: YTKNetwork 通过 YTKNetwork.h 管理其他类别，只需要在 .pch 导入 YTKNetwork.h 即可，YTKNetwork.h 代码如下：

```objectivec
#import <Foundation/Foundation.h>

#ifndef _YTKNETWORK_
    #define _YTKNETWORK_

#if __has_include(<YTKNetwork/YTKNetwork.h>)

    FOUNDATION_EXPORT double YTKNetworkVersionNumber;
    FOUNDATION_EXPORT const unsigned char YTKNetworkVersionString[];

    #import <YTKNetwork/YTKRequest.h>
    #import <YTKNetwork/YTKBaseRequest.h>
    #import <YTKNetwork/YTKNetworkAgent.h>
    #import <YTKNetwork/YTKBatchRequest.h>
    #import <YTKNetwork/YTKBatchRequestAgent.h>
    #import <YTKNetwork/YTKChainRequest.h>
    #import <YTKNetwork/YTKChainRequestAgent.h>
    #import <YTKNetwork/YTKNetworkConfig.h>

#else

    #import "YTKRequest.h"
    #import "YTKBaseRequest.h"
    #import "YTKNetworkAgent.h"
    #import "YTKBatchRequest.h"
    #import "YTKBatchRequestAgent.h"
    #import "YTKChainRequest.h"
    #import "YTKChainRequestAgent.h"
    #import "YTKNetworkConfig.h"

#endif /* __has_include */

#endif /* _YTKNETWORK_ */
```


##### 4.2.2 YTKBaseRequest 

功能：所有请求类的基类，持有 NSURLSessionTask 实例，responseData 等重要数据，提供了一些需要子类实现的与网络请求相关的放阿飞，处理回调的 block 和代理，命令 YTKNetworkAgent 发起网络请求。    

```objectivec
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

FOUNDATION_EXPORT NSString *const YTKRequestValidationErrorDomain;

NS_ENUM(NSInteger) {
    YTKRequestValidationErrorInvalidStatusCode = -8,
    YTKRequestValidationErrorInvalidJSONFormat = -9,
};

// HTTP 请求方式
typedef NS_ENUM(NSInteger, YTKRequestMethod) {
    YTKRequestMethodGET = 0,
    YTKRequestMethodPOST,
    YTKRequestMethodHEAD,
    YTKRequestMethodPUT,
    YTKRequestMethodDELETE,
    YTKRequestMethodPATCH,
};

// 请求数据序列化的方式 HTTP 还是 JSON
typedef NS_ENUM(NSInteger, YTKRequestSerializerType) {
    YTKRequestSerializerTypeHTTP = 0,
    YTKRequestSerializerTypeJSON,
};

// 返回数据的序列化方式，决定了 responseObject 的数据类型
typedef NS_ENUM(NSInteger, YTKResponseSerializerType) {
    -- NSData
    YTKResponseSerializerTypeHTTP,
    -- JSON 对象
    YTKResponseSerializerTypeJSON,
    -- NSXMLParser
    YTKResponseSerializerTypeXMLParser,
};

// 请求的优先级
typedef NS_ENUM(NSInteger, YTKRequestPriority) {
    YTKRequestPriorityLow = -4L,
    YTKRequestPriorityDefault = 0,
    YTKRequestPriorityHigh = 4,
};

// 声明了3个 block
@protocol AFMultipartFormData;

typedef void (^AFConstructingBlock)(id<AFMultipartFormData> formData);
typedef void (^AFURLSessionTaskProgressBlock)(NSProgress *);

@class YTKBaseRequest;

typedef void(^YTKRequestCompletionBlock)(__kindof YTKBaseRequest *request);

// 声明了YTKRequestDelegate 协议，定义了一系列可以用来接受网络相关的消息的方法，所有的代理方法将在主队列中调用
@protocol YTKRequestDelegate <NSObject>

@optional
// 请求成功结束
- (void)requestFinished:(__kindof YTKBaseRequest *)request;
// 请求失败
- (void)requestFailed:(__kindof YTKBaseRequest *)request;

@end

// YTKRequestAccessory 协议定义了一系列用来跟踪请求状态的方法，所有的代理方法将在主队列中调用
@protocol YTKRequestAccessory <NSObject>

@optional

// 请求即将开始
- (void)requestWillStart:(id)request;

// 请求即将结束（这个方法将在调用 requestFinished 和 successCompletionBlock 前执行）
- (void)requestWillStop:(id)request;

// 请求已经结束（这个方法将在调用 requestFinished 和 successCompletionBlock 后执行）
- (void)requestDidStop:(id)request;

@end

// YTKBaseRequest 是网络请求的抽象类，它提供了许多选项用于构建请求，是 YTKRequest 的基类
@interface YTKBaseRequest : NSObject

#pragma mark - Request and Response Information
///=============================================================================
/// @name Request and Response Information
///=============================================================================

// NSURLSessionTask 底层相关的

// 在请求开始之前这个值是空且不应该被访问
@property (nonatomic, strong, readonly) NSURLSessionTask *requestTask;

// 就是 requestTask.currentRequest
@property (nonatomic, strong, readonly) NSURLRequest *currentRequest;

// 就是 requestTask.originalRequest
@property (nonatomic, strong, readonly) NSURLRequest *originalRequest;

// 就是 requestTask.response
@property (nonatomic, strong, readonly) NSHTTPURLResponse *response;

///  The response status code.
@property (nonatomic, readonly) NSInteger responseStatusCode;

///  The response header fields.
@property (nonatomic, strong, readonly, nullable) NSDictionary *responseHeaders;

// 响应的数据表现形式，请求失败则是 nil
@property (nonatomic, strong, readonly, nullable) NSData *responseData;

// 响应的字符串表现形式，请求失败则是 nil
@property (nonatomic, strong, readonly, nullable) NSString *responseString;

///  This serialized response object. The actual type of this object is determined by
///  `YTKResponseSerializerType`. Note this value can be nil if request failed.
///
///  @discussion If `resumableDownloadPath` and DownloadTask is using, this value will
///              be the path to which file is successfully saved (NSURL), or nil if request failed.
// 
@property (nonatomic, strong, readonly, nullable) id responseObject;

// 如果设置响应序列化方式是 YTKResponseSerializerTypeJSON，这个就是响应结果序列化后的对象
@property (nonatomic, strong, readonly, nullable) id responseJSONObject;

// 请求序列化错误或者网络错误，默认是 nil
@property (nonatomic, strong, readonly, nullable) NSError *error;

// 请求任务是否已经取消（self.requestTask.state == NSURLSessionTaskStateCanceling）
@property (nonatomic, readonly, getter=isCancelled) BOOL cancelled;

// 请求任务是否在执行（self.requestTask.state == NSURLSessionTaskStateRunning）
@property (nonatomic, readonly, getter=isExecuting) BOOL executing;


#pragma mark - Request Configuration
///=============================================================================
/// @name Request Configuration
///=============================================================================

// tag 可以用来标识请求，默认是0
@property (nonatomic) NSInteger tag;

// userInfo 可以用来存储请求的附加信息，默认是 nil
@property (nonatomic, strong, nullable) NSDictionary *userInfo;

// 请求的代理，如果使用了 block 回调就可以忽略这个，默认为 nil
@property (nonatomic, weak, nullable) id<YTKRequestDelegate> delegate;

// 请求成功的回调，如果 block 存在并且 requestFinished 代理方法也实现了的话，两个都会被调用，先调用代理，再在主队列中调用block
@property (nonatomic, copy, nullable) YTKRequestCompletionBlock successCompletionBlock;

// 请求失败的回调，如果 block 存在并且 requestFailed 代理方法也实现了的话，两个都会被调用，先调用代理，再在主队列中调用 block
@property (nonatomic, copy, nullable) YTKRequestCompletionBlock failureCompletionBlock;

// 设置附加对象（这是什么鬼？）如果调用 addAccessory 来增加，这个数组会自动创建，默认是 nil
@property (nonatomic, strong, nullable) NSMutableArray<id<YTKRequestAccessory>> *requestAccessories;

// 可以用于在 POST 请求中需要时构造 HTTP body，默认是 nil
@property (nonatomic, copy, nullable) AFConstructingBlock constructingBodyBlock;

// 设置断点续传下载请求的地址，默认是 nil
// 在请求开始之前，路径上的文件将被删除。如果请求成功，文件将会自动保存到这个路径，否则响应将被保存到 responseData 和 responseString 中。为了实现这个工作，服务器必须支持 Range 并且响应需要支持`Last-Modified`和`Etag`，具体了解 NSURLSessionDownloadTask
@property (nonatomic, strong, nullable) NSString *resumableDownloadPath;

// 捕获下载进度，也可以看看 resumableDownloadPath
@property (nonatomic, copy, nullable) AFURLSessionTaskProgressBlock resumableDownloadProgressBlock;

// 设置请求优先级，在 iOS8 + 可用，默认是YTKRequestPriorityDefault = 0
@property (nonatomic) YTKRequestPriority requestPriority;

// 设置请求完成回调 block
- (void)setCompletionBlockWithSuccess:(nullable YTKRequestCompletionBlock)success
                              failure:(nullable YTKRequestCompletionBlock)failure;

// 清除请求回调 block
- (void)clearCompletionBlock;

// 添加遵循 YTKRequestAccessory 协议的请求对象，相关的 requestAccessories
- (void)addAccessory:(id<YTKRequestAccessory>)accessory;

#pragma mark - Request Action
// 将当前 self 网络请求加入请求队列，并且开始请求
- (void)start;

// 从请求队列中移除 self 网络请求，并且取消请求
- (void)stop;

// 使用带有成功失败 blcok 回调的方法开始请求(储存 block，调用 start)
- (void)startWithCompletionBlockWithSuccess:(nullable YTKRequestCompletionBlock)success
                                    failure:(nullable YTKRequestCompletionBlock)failure;

#pragma mark - Subclass Override
///=============================================================================
/// @name Subclass Override
///=============================================================================

// 请求成功后，在切换到主线程之前，在后台线程上调用。要注意，如果加载了缓存，则将在主线程上调用此方法，就像`request Complete Filter`一样。
- (void)requestCompletePreprocessor;
// 请求成功时会在主线程被调用
- (void)requestCompleteFilter;
// 请求成功后，在切换到主线程之前，在后台线程上调用。
- (void)requestFailedPreprocessor;
// 请求失败时会在主线程被调用
- (void)requestFailedFilter;

// 基础URL，应该只包含地址的主要地址部分，如 http://www.example.com
- (NSString *)baseUrl;
// 请求地址的URL,应该只包含地址的路径部分，如/v1/user。baseUrl 和requestUrl 使用[NSURL URLWithString:relativeToURL]进行连接。所以要正确返回。
// 如果 requestUrl 本身就是一个有效的 URL,将不再和 baseUrl 连接， baseUrl 将被忽略
- (NSString *)requestUrl;
// 可选的 CDN 请求地址
- (NSString *)cdnUrl;

// 设置请求超时时间，默认 60 秒.
// 如果使用了 resumableDownloadPath(NSURLSessionDownloadTask)，NSURLReques t的 timeoutInterval 将会被完全忽略，一个有效的设置超时时间的方法就是设置 NSURLSessionConfiguration 的 timeoutIntervalForResource 属性。
- (NSTimeInterval)requestTimeoutInterval;

// 设置请求的参数
- (nullable id)requestArgument;

// 重写这个方法可以在缓存时过滤请求中的某些参数
- (id)cacheFileNameFilterForRequestArgument:(id)argument;

// 设置 HTTP 请求方式
- (YTKRequestMethod)requestMethod;

// 设置请求数据序列化的方式
- (YTKRequestSerializerType)requestSerializerType;

// 设置请求数据序列化的方式. See also `responseObject`.
- (YTKResponseSerializerType)responseSerializerType;

// 用来 HTTP 授权的用户名和密码，应该返回@[@"Username", @"Password"]这种格式
- (nullable NSArray<NSString *> *)requestAuthorizationHeaderFieldArray;

// 附加的 HTTP 请求头
- (nullable NSDictionary<NSString *, NSString *> *)requestHeaderFieldValueDictionary;

// 用来创建完全自定义的请求，返回一个 NSURLRequest，忽略`requestUrl`, `requestTimeoutInterval`,`requestArgument`, `allowsCellularAccess`, `requestMethod`，`requestSerializerType`
- (nullable NSURLRequest *)buildCustomUrlRequest;

// 发送请求时是否使用 CDN
- (BOOL)useCDN;

// 是否允许请求使用蜂窝网络，默认是允许
- (BOOL)allowsCellularAccess;

// 验证 responseJSONObject 是否正确的格式化了
- (nullable id)jsonValidator;

// 验证 responseStatusCode 是否是有效的，默认是 code 在200-300之间是有效的
- (BOOL)statusCodeValidator;

@end

NS_ASSUME_NONNULL_END
```   


##### 4.2.3 YTKRequest     

功能：YTKBaseRequest 的子类。负责缓存的处理，请求前查询缓存；请求后写入缓存。

```objectivec
@interface YTKRequest : YTKBaseRequest
 
//表示当前请求，是否忽略本地缓存 responseData
@property (nonatomic) BOOL ignoreCache;
 
/// 返回当前缓存的对象
- (id)cacheJson;
 
/// 是否当前的数据从缓存获得
- (BOOL)isDataFromCache;
 
/// 返回是否当前缓存需要更新【缓存是否超时】
- (BOOL)isCacheVersionExpired;
 
/// 强制更新缓存【不使用缓存数据】
- (void)startWithoutCache;
 
/// 手动将其他请求的 JsonResponse 写入该请求的缓存
- (void)saveJsonResponseToCacheFile:(id)jsonResponse;
 
/// 子类重写方法【参数方法】
- (NSInteger)cacheTimeInSeconds;    //当前请求指定时间内，使用缓存数据
- (long long)cacheVersion;    //当前请求，指定使用版本号的缓存数据
- (id)cacheSensitiveData;   
 
@end
```

发现 YTKRequest 主要是对请求数据缓存方面的处理。再看 .m 实现文件：

```objectivec
//YTKRequest.m
- (void)start {
 
    //1. 如果忽略缓存 -> 请求
    if (self.ignoreCache) {
        [self startWithoutCache];
        return;
    }
 
    //2. 如果存在下载未完成的文件 -> 请求
    if (self.resumableDownloadPath) {
        [self startWithoutCache];
        return;
    }
 
    //3. 获取缓存失败 -> 请求
    if (![self loadCacheWithError:nil]) {
        [self startWithoutCache];
        return;
    }
 
    //4. 到这里，说明一定能拿到可用的缓存，可以直接回调了（因为一定能拿到可用的缓存，所以一定是调用成功的 block 和代理）
    _dataFromCache = YES;
 
    dispatch_async(dispatch_get_main_queue(), ^{
 
        //5. 回调之前的操作
        //5.1 缓存处理
        [self requestCompletePreprocessor];
 
        //5.2 用户可以在这里进行真正回调前的操作
        [self requestCompleteFilter];
 
        YTKRequest *strongSelf = self;
 
        //6. 执行回调
        //6.1 请求完成的代理
        [strongSelf.delegate requestFinished:strongSelf];
 
        //6.2 请求成功的 block
        if (strongSelf.successCompletionBlock) {
            strongSelf.successCompletionBlock(strongSelf);
        }
 
        //7. 把成功和失败的 block 都设置为 nil，避免循环引用
        [strongSelf clearCompletionBlock];
    });
}
```

通过 start() 方法可以看出，它做的是请求之前的查询和检查工作。下面是具体实现的过程：

（1）.ignoreCache 属性是用户手动设置的，如果用户强制忽略缓存，则无论是否缓存是否存在，直接发送请求。

（2）resumableDownloadPath 是断点下载路径，如果该路径不为空，说明有未完成的下载任务，则直接发送请求继续下载。

（3）loadCacheWithError：方法验证了加载缓存是否成功的方法（方法如果返回YES，说明可以加载缓存，反正则不可以加载缓存）下面是loadCacheWithError的具体实现：

```objectivec
- (BOOL)loadCacheWithError:(NSError * _Nullable __autoreleasing *)error {
 
    // 缓存时间小于0，则返回（缓存时间默认为-1，需要用户手动设置，单位是秒）
    if ([self cacheTimeInSeconds] < 0) {
        if (error) {
            *error = [NSError errorWithDomain:YTKRequestCacheErrorDomain code:YTKRequestCacheErrorInvalidCacheTime userInfo:@{ NSLocalizedDescriptionKey:@"Invalid cache time"}];
        }
        return NO;
    }
 
    // 是否有缓存的元数据，如果没有，返回错误（元数据是指数据的数据，在这里描述了缓存数据本身的一些特征：包括版本号，缓存时间，敏感信息等等）
    if (![self loadCacheMetadata]) {
        if (error) {
            *error = [NSError errorWithDomain:YTKRequestCacheErrorDomain code:YTKRequestCacheErrorInvalidMetadata userInfo:@{ NSLocalizedDescriptionKey:@"Invalid metadata. Cache may not exist"}];
        }
        return NO;
    }
 
    // 有缓存，再验证是否有效
    if (![self validateCacheWithError:error]) {
        return NO;
    }
 
    // 有缓存，而且有效，再验证是否能取出来
    if (![self loadCacheData]) {
        if (error) {
            *error = [NSError errorWithDomain:YTKRequestCacheErrorDomain code:YTKRequestCacheErrorInvalidCacheData userInfo:@{ NSLocalizedDescriptionKey:@"Invalid cache data"}];
        }
        return NO;
    }
    return YES;
}
```

上面代码标红的提到缓存的元数据的，.m 也实现类元数据的获取方法：loadCacheMetadata

```objectivec
//YTKRequest.m
- (BOOL)loadCacheMetadata {
 
    NSString *path = [self cacheMetadataFilePath];
    NSFileManager * fileManager = [NSFileManager defaultManager];
    if ([fileManager fileExistsAtPath:path isDirectory:nil]) {
        @try {
            //将序列化之后被保存在磁盘里的文件反序列化到当前对象的属性cacheMetadata
            _cacheMetadata = [NSKeyedUnarchiver unarchiveObjectWithFile:path];
            return YES;
        } @catch (NSException *exception) {
            YTKLog(@"Load cache metadata failed, reason = %@", exception.reason);
            return NO;
        }
    }
    return NO;
}
cacheMetadata（YTKCacheMetadata） 是当前 reqeust 类用来保存缓存元数据的属性。
YTKCacheMetadata 类被定义在 YTKRequest.m 文件里面：
 
//YTKRequest.m
@interface YTKCacheMetadata : NSObject@property (nonatomic, assign) long long version;
@property (nonatomic, strong) NSString *sensitiveDataString;
@property (nonatomic, assign) NSStringEncoding stringEncoding;
@property (nonatomic, strong) NSDate *creationDate;
@property (nonatomic, strong) NSString *appVersionString;
 
@end

//通过归档方式进行元数据的存储
- (void)encodeWithCoder:(NSCoder *)aCoder {
    [aCoder encodeObject:@(self.version) forKey:NSStringFromSelector(@selector(version))];
    [aCoder encodeObject:self.sensitiveDataString forKey:NSStringFromSelector(@selector(sensitiveDataString))];
    [aCoder encodeObject:@(self.stringEncoding) forKey:NSStringFromSelector(@selector(stringEncoding))];
    [aCoder encodeObject:self.creationDate forKey:NSStringFromSelector(@selector(creationDate))];
    [aCoder encodeObject:self.appVersionString forKey:NSStringFromSelector(@selector(appVersionString))];
}

- (nullable instancetype)initWithCoder:(NSCoder *)aDecoder {
    self = [self init];
    if (!self) {
        return nil;
    }

    self.version = [[aDecoder decodeObjectOfClass:[NSNumber class] forKey:NSStringFromSelector(@selector(version))] integerValue];
    self.sensitiveDataString = [aDecoder decodeObjectOfClass:[NSString class] forKey:NSStringFromSelector(@selector(sensitiveDataString))];
    self.stringEncoding = [[aDecoder decodeObjectOfClass:[NSNumber class] forKey:NSStringFromSelector(@selector(stringEncoding))] integerValue];
    self.creationDate = [aDecoder decodeObjectOfClass:[NSDate class] forKey:NSStringFromSelector(@selector(creationDate))];
    self.appVersionString = [aDecoder decodeObjectOfClass:[NSString class] forKey:NSStringFromSelector(@selector(appVersionString))];

    return self;
}
```

loadCacheMetadata 方法的目的是将之前被序列化保存的缓存元数据信息反序列化，赋给自身的 cacheMetadata 属性上。

现在获取了缓存的元数据并赋值给 cacheMetadata 属性上，接下来要元数据的各项信息是否符合要求：使用 validateCacheWithError：方法进行验证。

```objectivec
- (BOOL)validateCacheWithError:(NSError * _Nullable __autoreleasing *)error {
 
    // 是否大于过期时间
    NSDate *creationDate = self.cacheMetadata.creationDate;
    NSTimeInterval duration = -[creationDate timeIntervalSinceNow];
    if (duration < 0 || duration > [self cacheTimeInSeconds]) {
        if (error) {
            *error = [NSError errorWithDomain:YTKRequestCacheErrorDomain code:YTKRequestCacheErrorExpired userInfo:@{ NSLocalizedDescriptionKey:@"Cache expired"}];
        }
        return NO;
    }
 
    // 缓存的版本号是否符合
    long long cacheVersionFileContent = self.cacheMetadata.version;
    if (cacheVersionFileContent != [self cacheVersion]) {
        if (error) {
            *error = [NSError errorWithDomain:YTKRequestCacheErrorDomain code:YTKRequestCacheErrorVersionMismatch userInfo:@{ NSLocalizedDescriptionKey:@"Cache version mismatch"}];
        }
        return NO;
    }
 
    // 敏感信息是否符合
    NSString *sensitiveDataString = self.cacheMetadata.sensitiveDataString;
    NSString *currentSensitiveDataString = ((NSObject *)[self cacheSensitiveData]).description;
    if (sensitiveDataString || currentSensitiveDataString) {
        // If one of the strings is nil, short-circuit evaluation will trigger
        if (sensitiveDataString.length != currentSensitiveDataString.length || ![sensitiveDataString isEqualToString:currentSensitiveDataString]) {
            if (error) {
                *error = [NSError errorWithDomain:YTKRequestCacheErrorDomain code:YTKRequestCacheErrorSensitiveDataMismatch userInfo:@{ NSLocalizedDescriptionKey:@"Cache sensitive data mismatch"}];
            }
            return NO;
        }
    }
 
    // app 的版本是否符合
    NSString *appVersionString = self.cacheMetadata.appVersionString;
    NSString *currentAppVersionString = [YTKNetworkUtils appVersionString];
    if (appVersionString || currentAppVersionString) {
        if (appVersionString.length != currentAppVersionString.length || ![appVersionString isEqualToString:currentAppVersionString]) {
            if (error) {
                *error = [NSError errorWithDomain:YTKRequestCacheErrorDomain code:YTKRequestCacheErrorAppVersionMismatch userInfo:@{ NSLocalizedDescriptionKey:@"App version mismatch"}];
            }
            return NO;
        }
    }
    return YES;
}
```

如果每一项元数据信息都能通过，再在 loadCacheData 方法里面验证缓存是否能取出：

```objectivec
- (BOOL)loadCacheData {
 
    NSString *path = [self cacheFilePath];
    NSFileManager *fileManager = [NSFileManager defaultManager];
    NSError *error = nil;
 
    if ([fileManager fileExistsAtPath:path isDirectory:nil]) {
        NSData *data = [NSData dataWithContentsOfFile:path];
        _cacheData = data;
        _cacheString = [[NSString alloc] initWithData:_cacheData encoding:self.cacheMetadata.stringEncoding];
        switch (self.responseSerializerType) {
            case YTKResponseSerializerTypeHTTP:
                // Do nothing.
                return YES;
            case YTKResponseSerializerTypeJSON:
                _cacheJSON = [NSJSONSerialization JSONObjectWithData:_cacheData options:(NSJSONReadingOptions)0 error:&error];
                return error == nil;
            case YTKResponseSerializerTypeXMLParser:
                _cacheXML = [[NSXMLParser alloc] initWithData:_cacheData];
                return YES;
        }
    }
    return NO;
}
```

如果通过了最终的考验，则说明当前请求对应的缓存是符合各项要求并可以被成功取出，也就是可以直接进行回调了。当确认缓存可以成功取出后，手动设置 dataFromCache 属性为 YES，说明当前的请求结果是来自于缓存，而没有通过网络请求。

在这里面还有一个比较重要的方法：requestCompletePreprocessor

```objectivec
//YTKRequest.m：
- (void)requestCompletePreprocessor {
 
    [super requestCompletePreprocessor];
 
    //是否异步将 responseData 写入缓存（写入缓存的任务放在专门的队列 ytkrequest_cache_writing_queue 进行）
    if (self.writeCacheAsynchronously) {
 
        dispatch_async(ytkrequest_cache_writing_queue(), ^{
            //保存响应数据到缓存
            [self saveResponseDataToCacheFile:[super responseData]];
        });
 
    } else {
        //保存响应数据到缓存
        [self saveResponseDataToCacheFile:[super responseData]];
    }
}

//YTKRequest.m：
//保存响应数据到缓存
- (void)saveResponseDataToCacheFile:(NSData *)data {
 
    if ([self cacheTimeInSeconds] > 0 && ![self isDataFromCache]) {
        if (data != nil) {
            @try {
                // New data will always overwrite old data.
                [data writeToFile:[self cacheFilePath] atomically:YES];
 
                YTKCacheMetadata *metadata = [[YTKCacheMetadata alloc] init];
                metadata.version = [self cacheVersion];
                metadata.sensitiveDataString = ((NSObject *)[self cacheSensitiveData]).description;
                metadata.stringEncoding = [YTKNetworkUtils stringEncodingWithRequest:self];
                metadata.creationDate = [NSDate date];
                metadata.appVersionString = [YTKNetworkUtils appVersionString];
                [NSKeyedArchiver archiveRootObject:metadata toFile:[self cacheMetadataFilePath]];
 
            } @catch (NSException *exception) {
                YTKLog(@"Save cache failed, reason = %@", exception.reason);
            }
        }
    }
}
```

我们可以看到, requestCompletePreprocessor 方法的任务是将响应数据保存起来，也就是做缓存。但是，缓存的保存有两个条件，一个是需要 cacheTimeInSeconds 方法返回正整数（缓存时间，单位是秒）；另一个条件就是 isDataFromCache 方法返回 NO 。

进一步研究：startWithoutCache

这个方法做了哪些？

```objectivec
//YTKRequest.m
- (void)startWithoutCache {
 
    //1. 清除缓存
    [self clearCacheVariables];
 
    //2. 调用父类的发起请求
    [super start];
}




//YTKBaseRequest.m:
- (void)start {
 
    //1. 告诉 Accessories 即将回调了（其实是即将发起请求）
    [self toggleAccessoriesWillStartCallBack];
 
    //2. 令 agent 添加请求并发起请求，在这里并不是组合关系，agent 只是一个单例
    [[YTKNetworkAgent sharedAgent] addRequest:self];
}
```


##### 4.2.4 YTKNetworkConfig 

功能：被 YTKRequest 和 YTKNetworkAgent 访问。负责所有请求的全局配置，对于 baseUrl 和 CDNUrl 等等。

在实际业务中，作为公司的测试需要不断的切换服务器地址，确定数据的正确性，也间接说明 YTKNetworkConfig 的必要性。    

下面是以自己目前公司所用的 YTKNetwork 的 YTKNetworkConfig 的用处：

（1）首先在 AppDelegate 中：- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions 方法配置服务：

```objectivec
- (BOOL)application:(UIApplication *)applicationdidFinishLaunchingWithOptions:(NSDictionary *)launchOptinos {
#ifdef DEBUG
    // test
    //[RLPickerViewModel parserAddressJson:nil];
    
    //配置日志
    //[self redirectNSLogToDocumentFolder];
#endif

   //配置请求服务
   [self setupServer];
}
```

紧接着：[self setupServer];

```objectivec
// 设置后 server
- (void)setupServer {
    [IOAApiManager configNetwork];
    [IOAApiManager startNetworkMonitoring];
    [self addNotification];
}
```

在 IOAApiManager 实现类方法：

```objectivec
+ (void)configwork {
    YTKNetworkAgent *agent = [YTKNetworkAgent sharedAgent];
    [agent setValue:[NSSet setWithObjects:@"application/json",@"text/json",@"text/javascript",@"text/html",@"text/plain",@"application/x-www-form-urlencodem",nil] forKeyPath:@"_manager.responseSerializer.acceptableContentTypes"];
    
    static ServerType serverType = kSeverTypeRelese;
    YTKNetwor
    kConfig *config = [YTKentworkConfig sharedConfig;
    switch (serverType){
        case kSeverTypeDev:    //开发服务器地址
            config.baseUrl = @"http://xxxxx.com/xxx"
            break;
        case kSeverTypeTest:    //测试服务器地址
            config.baseUrl = @"http://xxxxx.com/xxx"
            break;
        case kSeverTypeRelease:    //发布版服务器地址
            config.baseUrl = @"http://xxxxx.com/xxx"
            break;
        default:
            break;
    }
    //证书配置
    [self configHttps];
}
```

我们以后就可以通过 baseUrl 进行不同的地址访问啦。大部分企业可能需要一些静态资源（例如图片，js，css 等）这就使用到了 CDN，YTKNetworkConfig 的cdnUrl参数用于统一设置这一部分网络请求的地址。

YTKNetworkConfig 源码也提供了安全策略，url 过滤，缓存路径的过滤等方法如下:

```objectivec
// 使用类方法创建单例对象
+ (YTKNetworkConfig *)sharedConfig;

// 请求的根 URL，默认是空字符串
@property (nonatomic, strong) NSString *baseUrl;

// 请求 CDN URL，默认是空字符串
@property (nonatomic, strong) NSString *cdnUrl;

// URL 过滤池（YTKUrlFilterProtocol 协议使用）
@property (nonatomic, strong, readonly) NSArray<id<YTKUrlFilterProtocol>> *urlFilters;

// 缓存路径的过滤池（YTKCacheDirPathFilterProtocol 协议使用）
@property (nonatomic, strong, readonly) NSArray<id<YTKCacheDirPathFilterProtocol>> *cacheDirPathFilters;

// 同 AFNetworking 中使用的安全策略
@property (nonatomic, strong) AFSecurityPolicy *securityPolicy;

// 是否记录调试信息，默认是 NO
@property (nonatomic) BOOL debugLogEnabled;

// 用来初始化 AFHTTPSessionManager,默认是 nil
@property (nonatomic, strong) NSURLSessionConfiguration* sessionConfiguration;

// 添加一个新的 URL 过滤器
- (void)addUrlFilter:(id<YTKUrlFilterProtocol>)filter;

// 删除所有的 URL 过滤器
- (void)clearUrlFilter;

// 添加一个新的缓存地址过滤器
- (void)addCacheDirPathFilter:(id<YTKCacheDirPathFilterProtocol>)filter;


//删除所有的缓存地址过滤器
 - (void)clearCacheDirPathFilter; @end
```


##### 4.2.5 YTKNetworkAgent 

功能：真正发起请求的类，负责发起请求，结束请求，并持有一个字典来存储正在执行的请求。
    
1、首先从请求开始：YTKNetworkAgent 把当前的请求对象添加到了自己身上并发送请求。  
  
```objectivec
//YTKNetworkAgent.m
- (void)addRequest:(YTKBaseRequest *)request {
 
    //1. 获取 task
    NSParameterAssert(request != nil);
 
    NSError * __autoreleasing requestSerializationError = nil;
 
    //获取用户自定义的 requestURL
    NSURLRequest *customUrlRequest= [request buildCustomUrlRequest];
 
    if (customUrlRequest) {
 
        __block NSURLSessionDataTask *dataTask = nil;
        //如果存在用户自定义 request，则直接走 AFNetworking 的 dataTaskWithRequest:方法
        dataTask = [_manager dataTaskWithRequest:customUrlRequest completionHandler:^(NSURLResponse * _Nonnull response, id  _Nullable responseObject, NSError * _Nullable error) {
            //响应的统一处理
            [self handleRequestResult:dataTask responseObject:responseObject error:error];
        }];
        request.requestTask = dataTask;
 
    } else {
 
        //如果用户没有自定义 url，则直接走这里
        request.requestTask = [self sessionTaskForRequest:request error:&requestSerializationError];
 
    }
 
    //序列化失败，则认定为请求失败
    if (requestSerializationError) {
        //请求失败的处理
        [self requestDidFailWithRequest:request error:requestSerializationError];
        return;
    }
 
    NSAssert(request.requestTask != nil, @"requestTask should not be nil");
 
    // 优先级的映射
    // !!Available on iOS 8 +
    if ([request.requestTask respondsToSelector:@selector(priority)]) {
        switch (request.requestPriority) {
            case YTKRequestPriorityHigh:
                request.requestTask.priority = NSURLSessionTaskPriorityHigh;
                break;
            case YTKRequestPriorityLow:
                request.requestTask.priority = NSURLSessionTaskPriorityLow;
                break;
            case YTKRequestPriorityDefault:
                /*!!fall through*/
            default:
                request.requestTask.priority = NSURLSessionTaskPriorityDefault;
                break;
        }
    }
 
    // Retain request
    YTKLog(@"Add request: %@", NSStringFromClass([request class]));
 
    //2. 将request放入保存请求的字典中，taskIdentifier 为 key，request 为值
    [self addRequestToRecord:request];
 
    //3. 开始 task
    [request.requestTask resume];
}
```

这个方法可以看出调用了 AFNetworking 的请求方法，也验证了 YTKNetwork 对 AFNetworking 高度封装。这个方法可以分成三部分：

(1).获取当前请求对应的 task 并赋值给 request 的 requestTask 属性

(2).将 request 放入专门用来保存请求的字典中，key 为 taskIdentifier

(3).启动 task


#### 4.2.6 YTKBatchRequest 

功能介绍:可以发起批量请求，持有一个数组来保存所有的请求类。在请求执行后遍历这个数组发起请求，如果其中有一个请求返回失败，则认定本组请求失败。    
    
下面看一个初始化方法:    
	
```objectivec
//YTKBatchRequest.m
- (instancetype)initWithRequestArray:(NSArray*)requestArray {
    self = [super init];
    if (self) {
 
        //保存为属性
        _requestArray = [requestArray copy];
 
        //批量请求完成的数量初始化为0
        _finishedCount = 0;
 
        //类型检查，所有元素都必须为YTKRequest或的它的子类，否则强制初始化失败
        for (YTKRequest * req in _requestArray) {
            if (![req isKindOfClass:[YTKRequest class]]) {
                YTKLog(@"Error, request item must be YTKRequest instance.");
                return nil;
            }
        }
    }
    return self;
}
```

初始化以后，我们就可以调用 start 方法来发起当前 YTKBatchRequest 实例所管理的所有请求了。

```objectivec
- (void)start {
 
    //如果batch里第一个请求已经成功结束，则不能再start
    if (_finishedCount > 0) {
        YTKLog(@"Error! Batch request has already started.");
        return;
    }
 
    //最开始设定失败的request为nil
    _failedRequest = nil;
 
    //使用YTKBatchRequestAgent来管理当前的批量请求
    [[YTKBatchRequestAgent sharedAgent] addBatchRequest:self];
    [self toggleAccessoriesWillStartCallBack];
 
    //遍历所有request，并开始请求
    for (YTKRequest * req in _requestArray) {
        req.delegate = self;
        [req clearCompletionBlock];
        [req start];
    }
}
```
所以在这里是遍历 YTKBatchRequest 实例的 _requestArray 并逐一发送请求。因为已经封装好了单个的请求，所以在这里直接start就好了。

（1）在请求之后，在每一个请求的回调的代理方法里面，来判断这次请求是否是成功的，也是在 YTKRequest 子类 YTKBatchRequest.m 判断。

```objectivec
//YTKBatchRequest.m
#pragma mark - Network Request Delegate
- (void)requestFinished:(YTKRequest *)request {
 
    //某个request成功后，首先让_finishedCount + 1
    _finishedCount++;
 
    //如果_finishedCount等于_requestArray的个数，则判定当前batch请求成功
    if (_finishedCount == _requestArray.count) {
 
        //调用即将结束的代理
        [self toggleAccessoriesWillStopCallBack];
 
        //调用请求成功的代理
        if ([_delegate respondsToSelector:@selector(batchRequestFinished:)]) {
            [_delegate batchRequestFinished:self];
        }
 
        //调用批量请求成功的block
        if (_successCompletionBlock) {
            _successCompletionBlock(self);
        }
 
        //清空成功和失败的block
        [self clearCompletionBlock];
 
        //调用请求结束的代理
        [self toggleAccessoriesDidStopCallBack];
 
        //从YTKBatchRequestAgent里移除当前的batch
        [[YTKBatchRequestAgent sharedAgent] removeBatchRequest:self];
    }
}
```

在某个请求的回调成功以后，会让成功计数+1。在+1以后，如果成功计数和当前批量请求数组里元素的个数相等，则判定当前批量请求成功，并进行当前批量请求的成功回调。

（2）失败回调

```objectivec
//YTKBatchRequest.m
- (void)requestFailed:(YTKRequest *)request {
 
    _failedRequest = request;
 
    //调用即将结束的代理
    [self toggleAccessoriesWillStopCallBack];
 
    //停止batch里所有的请求
    for (YTKRequest *req in _requestArray) {
        [req stop];
    }
 
    //调用请求失败的代理
    if ([_delegate respondsToSelector:@selector(batchRequestFailed:)]) {
        [_delegate batchRequestFailed:self];
    }
 
    //调用请求失败的block
    if (_failureCompletionBlock) {
        _failureCompletionBlock(self);
    }
 
    //清空成功和失败的block
    [self clearCompletionBlock];
 
    //调用请求结束的代理
    [self toggleAccessoriesDidStopCallBack];
 
    //从YTKBatchRequestAgent里移除当前的batch
    [[YTKBatchRequestAgent sharedAgent] removeBatchRequest:self];
}
```

从上面代码可以看出，如果批量请求里面有一个request失败了，则判定当前批量请求失败。



##### 4.2.7 YTKBatchRequestAgent 

功能简介：负责管理多个YTKBatchRequest实例，持有一个数组保存YTKBatchRequest。支持添加和删除YTKBatchRequest实例。

YTKBatchRequestAgent和YTKRequestAgent差不多，只不过是一个和多个区别，没有多少要核心讲的，可以自己看一下YTKBatchRequestAgent的源码，相信都能看懂。


##### 4.2.8 YTKChainRequest     

功能简介：可以发起链式请求，持有一个数组来保存所有的请求类。当某个请求结束后才能发起下一个请求，如果其中有一个请求返回失败，则认定本请求链失败。（链式请求：例如：发送请求 A，根据请求 A 的结果，选择性的发送请求 B 和 C，再根据 B 和 C 的结果，选择性的发送请求 D。）

处理链式请求的类是 YTKChainRequest，利用 YTKChainRequestAgent 单例来管理 YTKChainRequest 实例。

初始化方法:   
	
```objectivec
//YTKChainRequest.m
- (instancetype)init {
 
    self = [super init];
    if (self) {
 
        //下一个请求的 index
        _nextRequestIndex = 0;
 
        //保存链式请求的数组
        _requestArray = [NSMutableArray array];
 
        //保存回调的数组
        _requestCallbackArray = [NSMutableArray array];
 
        //空回调，用来填充用户没有定义的回调 block
        _emptyCallback = ^(YTKChainRequest *chainRequest, YTKBaseRequest *baseRequest) {
            // do nothing
        };
    }
    return self;
}
```

YTKChainRequest 提供了添加和删除 request 的接口。

```objectivec
//在当前 chain 添加 request 和 callback
- (void)addRequest:(YTKBaseRequest *)request callback:(YTKChainCallback)callback {
 
    //保存当前请求
    [_requestArray addObject:request];
 
    if (callback != nil) {
        [_requestCallbackArray addObject:callback];
    } else {
        //之所以特意弄一个空的 callback，是为了避免在用户没有给当前 request 的 callback 传值的情况下，造成 request 数组和 callback 数组的不对称
        [_requestCallbackArray addObject:_emptyCallback];
    }
}
```

（1）链式请求发起

```objectivec
//YTKChainRequest.m
- (void)start {
    //如果第1个请求已经结束，就不再重复start了
    if (_nextRequestIndex > 0) {
        YTKLog(@"Error! Chain request has already started.");
        return;
    }
    //如果请求队列数组里面还有request，则取出并start
    if ([_requestArray count] > 0) {
        [self toggleAccessoriesWillStartCallBack];
        //取出当前request并start
        [self startNextRequest];
        //在当前的_requestArray添加当前的chain（YTKChainRequestAgent允许有多个chain）
        [[YTKChainRequestAgent sharedAgent] addChainRequest:self];
    } else {
        YTKLog(@"Error! Chain request array is empty.");
    }
}
```

通过查看链式请求的实现，发现链式请求的请求队列是可以变动的，用户可以无限制地添加请求。只要请求队列里面有请求存在，则YTKChainRequest就会继续发送它们。

（2）链式请求的请求和回调

下面是终止方法stop()

```objectivec
//YTKChainRequest.m
//终止当前的chain
- (void)stop {
 
    //首先调用即将停止的callback
    [self toggleAccessoriesWillStopCallBack];
 
    //然后stop当前的请求，再清空chain里所有的请求和回掉block
    [self clearRequest];
 
    //在YTKChainRequestAgent里移除当前的chain
    [[YTKChainRequestAgent sharedAgent] removeChainRequest:self];
 
    //最后调用已经结束的callback
    [self toggleAccessoriesDidStopCallBack];
}
```

stop方法是可以在外部调用的，所以用户可以随时终止当前链式请求的进行。它首先调用clearReuqest方法，将当前request停止，再将请求队列数组和callback数组清空。

```objectivec
//YTKChainRequest.m
- (void)clearRequest {
    //获取当前请求的index
    NSUInteger currentRequestIndex = _nextRequestIndex - 1;
    if (currentRequestIndex < [_requestArray count]) {
        YTKBaseRequest *request = _requestArray[currentRequestIndex];
        [request stop];
    }
    [_requestArray removeAllObjects];
    [_requestCallbackArray removeAllObjects];
}
```

然后在YTKChainRequestAgent单例里面，将自己移除掉。


##### 4.2.9 YTKNetworkPrivate 

功能简介：提供JSON验证，appVersion等辅助性的方法；给YTKBaseRequest增加一些分类。


### 5.杂谈：包括优缺点、配置方式、维护更新等。

#### 5.1、优缺点：    
优点：方便更换底层网络库，方便在基类中处理缓存逻辑，以及其它一些公共逻辑，方便做对象持久化，维护更新方便    

缺点：不适合于在网络请求简单的项目中使用，这样显得没有直接使用原生的网络库使用方便。    

#### 5.2、配置方式：
将YTKNetwork 库导入工程中无需做任何配置，具体使用详见（1.2）。
#### 5.3 维护更新
1、CocoaPods 方式：在Podfile 文件中加入一行代码来使用YTKNetwork    
		
	pod 'YTKNetwork'

2、非CocoaPods 方式：github官网上下载最新版YTKNetwork库将AFNetworking和YTKnetwork 库抽取出来放到项目中。    

	


