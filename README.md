# YTKNetworkAnalysis



## YTKNetwork 源码分析


### 1. 功能介绍   

#### 1.1 YTKNetwork
YTKNetwork 基于AFNetWorking 封装的一个网络请求库，相比AFNetworking，YTKNetwork 提供了以下更高级的功能:    

 * 支持按时间缓存网络请求内容
 * 支持按版本号缓存网络请求内容
 * 支持统一设置服务器和 CDN 的地址
 * 支持检查返回 JSON 内容的合法性
 * 支持文件的断点续传
 * 支持 block 和 delegate 两种模式的回调方式
 * 支持批量的网络请求发送，并统一设置它们的回调（实现在YTKBatchRequest类中）
 * 支持方便地设置有相互依赖的网络请求的发送，例如：发送请求A，根据请求A的结果，选择性的发送请求B和C，再根据B和C的结果，选择性的发送请求D。（实现在YTKChainRequest类中）
 * 支持网络请求 URL 的 filter，可以统一为网络请求加上一些参数，或者修改一些路径
 * 定义了一套插件机制，可以很方便地为 YTKNetwork 增加功能。猿题库官方现在提供了一个插件，可以在某些网络请求发起时，在界面上显示"正在加载"的 HUD
    
#### 1.2  YTKNetwork 的基本使用
##### Step 1: 如需统一为网络请求接口加上一些参数和设置统一的服务器，需要在入口类的application didFinishLaunchingWithOptions:中进行如下配置：    

	- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    	NSString *appVersion = [[[NSBundle mainBundle] infoDictionary] objectForKey:@"CFBundleShortVersionString"];
    	//设置统一的服务器接口，可通过如下的baseUrl 进行设置（也可不同过详见下文）如不需要，可忽略
    	YTKNetworkConfig *config = [YTKNetworkConfig sharedInstance];
    	config.baseUrl = @"http://api.alilo.com.cn";
    	// Filter 表示统一为所有的请求添加一个或多个参数(例如统一添加版本号参数)  （如不需要，可忽略）		
    	YTKUrlArgumentsFilter *urlFilter = [YTKUrlArgumentsFilter filterWithArguments:@{@"version": appVersion}];
    	[config addUrlFilter:urlFilter];
    	return YES;
	}

##### Step 2:构建请求对象继承于重写父类，并重写父类的方法填写参数。
例：修改请求接口需要重写父类的requestUrl 方法   
	
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
例:    

	TestGETRequest *test = [[TestGETRequest alloc] init];
    [test startWithCompletionBlockWithSuccess:^(__kindof YTKBaseRequest *request) {
        NSLog(@" responseJSONObject ==  \n %@",request.responseJSONObject);
    } failure:^(__kindof YTKBaseRequest *request) {
        NSLog(@"requestOperationError == %@",request.requestOperationError);    
    }];

相关教程和Demo可参考

 * [基础教程](https://github.com/yuantiku/YTKNetwork/blob/master/BasicGuide.md)
 * [高级教程](https://github.com/yuantiku/YTKNetwork/blob/master/ProGuide.md)
  

### 2. 总体设计(图文)   

#### 2.1总体结构图
![](/pics/flowchart.png )
上面是UIL的总体设计图，整个库分为YTKRequest，YTKNetworkAgent,YTKNetworkConfig和YTKNetworkPrivate四个模块。    

简而言之，对于我们项目中所有的网络请求结果，当数据从AFNetWorking 回调回来时，YTKNetworkAgent先对它进行JSON的合法性的校验和版本号缓存，最后再交给最初创建的YTKRequest对象进行处理。    
    
#### 2.2
UIL 中的概念(详见4.2)    

 *  YTKRequest: 负责网络请求根据版本号和时间进行缓存的网络请求基类，继承于YTKBaseRequest我们的网络请求如果需要缓存功能都需要继承于YTKRequest；    
 * YTKBatchRequest: 批量网络请求基类；    
 * YTKChainRequest: 链式请求基类；    
 * YTKNetworkAgent: 负责管理所有的网络请求；    
 * YTKNetworkConfig: 负责统一为网络请求加上一些参数，或者修改一些路径；   
 * YTKNetworkPrivate: 负责JSON 数据合法性的检查,URL的拼接以及加密等功能。
 

### 3. 流程图
YTKNewWork 网络请求库中最核心的两步：请求过程处理，以及请求到数据时的处理

图3.1    
![](/picx/request_start.png)    
上图为YTKNetwork 网络开始请求过程流程图：    
下图为YTKNetWork 处理返回结果流程图：    
图3.2    
![](/picx/new.png)


### 4. 详细设计

#### 4.1类关系UML
![](/picx/iOS网络请求库UML.png)    

#### 4.2  核心类功能介绍

目录：    

[1、YTKBaseRequest]()    
[2、YTKRequest]()    
[3、YTKNetworkAgent]()    
[4、YTKNetworkConfig]()    
[5、YTKNetworkPrivate]()    
[6、YTKBatchRequest]()    
[7、YTKBatchRequestAgent]()    
[8、YTKChainRequest]()    
[9、YTKChainRequestAgent]()


##### 4.2.1 YTKBaseRequest    

功能: YTKNetwork 网络请求基类，提供YTK网络请求类的基本操作与接口。    

自定义类型：    
1、YTKRequestMethod HTTP 请求的方法    

	typedef NS_ENUM(NSInteger , YTKRequestMethod) {    
    	YTKRequestMethodGet = 0,    
    	YTKRequestMethodPost,
    	YTKRequestMethodHead,
    	YTKRequestMethodPut,
    	YTKRequestMethodDelete,
    	YTKRequestMethodPatch,
	};

2、YTKRequestSerializerType    
		
	typedef NS_ENUM(NSInteger , YTKRequestSerializerType) {
    	YTKRequestSerializerTypeHTTP = 0,
    	YTKRequestSerializerTypeJSON,
	};
	
3、YTKRequestPriority 请求优先级    
		
	typedef NS_ENUM(NSInteger , YTKRequestPriority) {
    	YTKRequestPriorityLow = -4L,
    	YTKRequestPriorityDefault = 0,
    	YTKRequestPriorityHigh = 4,
	};	
4、AFConstructingBlock    

	typedef void (^AFConstructingBlock)(id<AFMultipartFormData> formData);
	
5、AFDownloadProgressBlock 下载的进度回调    
	
	typedef void (^AFDownloadProgressBlock)(AFDownloadRequestOperation *operation, NSInteger bytesRead, long long totalBytesRead, long long totalBytesExpected, long long totalBytesReadForFile, long long totalBytesExpectedToReadForFile);
	
6、YTKRequestCompletionBlock 请求完成回调包括成功或者失败    
	
	typedef void(^YTKRequestCompletionBlock)(__kindof YTKBaseRequest *request);
    
    
    
协议（接口）：    

1、YTKRequestDelegate 普通请求协议    

	@protocol YTKRequestDelegate <NSObject>

	@optional

	- (void)requestFinished:(YTKBaseRequest *)request;   
	- (void)requestFailed:(YTKBaseRequest *)request;
	- (void)clearRequest;

	@end

2、YTKRequestAccessory 以自身子请求相关    

	@protocol YTKRequestAccessory <NSObject>

	@optional

	- (void)requestWillStart:(id)request;
	- (void)requestWillStop:(id)request;
	- (void)requestDidStop:(id)request;
	
	@end


主要成员对象:     

1、requestOperation：该对象完成YTKNetwork与AFNetWorking之间的过度，所有的请求最终都转为requestOperation来完成；   
2、delegate：id\<YTKRequestDelegate\>外部收听的回调接口    
3、successCompletionBlock：（YTKRequestCompletionBlock）成功回调的Block;    
4、failureCompletionBlock：（YTKRequestCompletionBlock）请求失败的回调block;    
5、requestAccessories：（NSMutableArray *）YTKbaseRequest附属的请求(基本的请求用不到，详细使用见YTKChainRequest,和YTKBatchRequest)    

主要方法（函数）：    
1、- (NSString *)requestUrl; 请求路径（子类实现）,若不希望使用统一服务器地址，则在此方法内输入完整的URL    
2、- (NSString *)cdnUrl，返回CDN服务器地址（如果有）;    
3、- (NSString *)baseUrl，服务器接口地址，如果统一设置了服务器地址，则统一设置的会被该baseURL返回的接口覆盖掉；    
4、- (NSTimeInterval)requestTimeoutInterval，设置超时；    
5、- (id)requestArgument，请求的参数该方法内填写请求的参数；    
6、- (YTKRequestMethod)requestMethod，请求方法，须子类填写；    
7、- (BOOL)statusCodeValidator，检验状态码的合法性；    
8、- (NSString *)resumableDownloadPath，断点续传路径；    
9、- (void)start，请求开始
10、- (void)stop，请求停止
11、- (void)setCompletionBlockWithSuccess:(YTKRequestCompletionBlock)success failure:(YTKRequestCompletionBlock)failure 设置请求成功失败回调；    
11、- (void)startWithCompletionBlockWithSuccess:(YTKRequestCompletionBlock)success failure:(YTKRequestCompletionBlock)failure 请求开始并设置成功和失败的回调；    


##### 4.2.2 YTKRequest 类

功能：继承了YTKBaseRequest 的所有功能，此外增加了对缓存的处理，是YTK所有增加缓存功能请求的基类，我们项目中需要缓存的请求类都需要直接或间接继承与它。    

主要成员变量：    

1、ignoreCache ：BOOL 类型，默认为不忽略，

主要函数：（方法）  
1、- (id)cacheJson，该方法返回当前缓存的对象；    
2、- (BOOL)isDataFromCache，是否当前的数据从缓存获得；    
3、- (BOOL)isCacheVersionExpired， 返回是否当前缓存需要更新；    
4、- (void)startWithoutCache，强制更新缓存；    
5、- (void)saveJsonResponseToCacheFile:(id)jsonResponse，保存缓存，官方解释为：手动将其他请求的JsonResponse写入该请求的缓存，比如AddNoteApi, UpdateNoteApi都会获得Note，且其与GetNoteApi共享缓存，可以通过这个接口写入GetNoteApi缓存    
		
	- (void)saveJsonResponseToCacheFile:(id)jsonResponse {
    	if ([self cacheTimeInSeconds] > 0 && ![self isDataFromCache]) {
        	NSDictionary *json = jsonResponse;
        	if (json != nil) {
            	[NSKeyedArchiver archiveRootObject:json toFile:[self cacheFilePath]];
	          [NSKeyedArchiver archiveRootObject:@([self cacheVersion]) toFile:[self cacheVersionFilePath]];
        	}
    	}
	}



##### 4.2.3 YTKNetworkAgent 类    

功能：普通请求的管理类    

主要函数（函数）:    

1、+ (YTKNetworkAgent *)sharedInstance 获取、创建单例，初始化一些引用的对象；    
2、- (void)addRequest:(YTKBaseRequest *)request， 该方法根据Request的细节结合AFHTTPRequestOperation进行请求；    
3、- (void)cancelRequest:(YTKBaseRequest *)request，取消请操作；    
4、- (void)cancelAllRequests，取消所有请求；    
5、- (NSString *)buildRequestUrl:(YTKBaseRequest *)request，根据request和networkConfig构建url    
6、- (void)addOperation:(YTKBaseRequest *)request ，- (void)removeOperation:(AFHTTPRequestOperation *)operation 添加和移除请求操作，通过移除字典，特点：速度快；    
	
	- (void)addOperation:(YTKBaseRequest *)request {
    	if (request.requestOperation != nil) {
        	NSString *key = [self requestHashKey:request.requestOperation];
        	@synchronized(self) {
          		_requestsRecord[key] = request;
          	}
    	}
	}
	- (void)removeOperation:(AFHTTPRequestOperation *)operation {
    	NSString *key = [self requestHashKey:operation];
    	@synchronized(self) {
        	[_requestsRecord removeObjectForKey:key];
    	}
    	YTKLog(@"Request queue size = %lu", (unsigned long)[_requestsRecord count]);
	}



##### 4.2.4 YTKNetworkConfig 类

功能：统一为网络请求设置和修改一些参数，或修改一些路径。    

协议和接口：    
1、 YTKUrlFilterProtocol    

		@protocol YTKUrlFilterProtocol <NSObject>
		- (NSString *)filterUrl:(NSString *)originUrl withRequest:(YTKBaseRequest *)request;
		@end
		
2、YTKCacheDirPathFilterProtocol    

		@protocol YTKCacheDirPathFilterProtocol <NSObject>
		- (NSString *)filterCacheDirPath:(NSString *)originPath withRequest:(YTKBaseRequest *)request;
		@end

主要成员变量：    
1、baseUrl:(NSString *),统一设置的服务器地址，一旦设置，所有的请求都会默认使用该服务器地址，除非请求类内部重写了该方法或改变了将requestUrl完整化；    
2、cndUrl:(NSString *),统一设置的cnd服务器地址，同上；    
3、urlFilters：（NSArray *）,统一为服务器参数添加的所有字段；
4、cacheDirPathFilters：（NSArray *）需要修改的缓存路径集合；    
5、securityPolicy:(AFSecurityPolicy *),集成了AFNetworking 的安全策略，详细内容有待研究。


主要函数（方法）：   
1、- (void)addUrlFilter:(id<YTKUrlFilterProtocol>)filter，通过该方法统一为网络请求添加参数。    
2、- (void)addCacheDirPathFilter:(id <YTKCacheDirPathFilterProtocol>)filter，通过该方法统一修改缓存文件夹路径。


##### 4.2.5 YTKNetworkPrivate 类

功能：该类为一工具类，提供一些参数加密和JSON数据检查的接口。

主要方法：    
1、+ (BOOL)checkJson:(id)json withValidator:(id)validatorJson ，根据validatorJson描述的类型检查json数据是否符合规范，通过不断的递归校验最终返回json数据的正确性，源码如下：  
  
	+ (BOOL)checkJson:(id)json withValidator:(id)validatorJson {
    	if ([json isKindOfClass:[NSDictionary class]] &&
        	[validatorJson isKindOfClass:[NSDictionary class]]) {
        	NSDictionary * dict = json;
        	NSDictionary * validator = validatorJson;
        	BOOL result = YES;
        	NSEnumerator * enumerator = [validator keyEnumerator];
       		NSString * key;
        	while ((key = [enumerator nextObject]) != nil) {
            	id value = dict[key];
            	id format = validator[key];
            	if ([value isKindOfClass:[NSDictionary class]]
                || [value isKindOfClass:[NSArray class]]) {
                result = [self checkJson:value withValidator:format];
                if (!result) {
                    break;
                }
            } else {
                if ([value isKindOfClass:format] == NO &&
                    [value isKindOfClass:[NSNull class]] == NO) {
                    result = NO;
                    break;
                }
            }
        	}
        	return result;
    	} else if ([json isKindOfClass:[NSArray class]] &&
               [validatorJson isKindOfClass:[NSArray class]]) {
        	NSArray * validatorArray = (NSArray *)validatorJson;
        	if (validatorArray.count > 0) {
            	NSArray * array = json;
            	NSDictionary * validator = validatorJson[0];
            	for (id item in array) {
                	BOOL result = [self checkJson:item withValidator:validator];
                	if (!result) {
                   	return NO;
                	}
            	}
        	}
        	return YES;
    	} else if ([json isKindOfClass:validatorJson]) {
        	return YES;
    	} else {
        	return NO;
    	}
	}
2、+ (NSString *)urlStringWithOriginUrlString:(NSString *)originUrlString
                          appendParameters:(NSDictionary *)parameters ，拼接URL；    
3、+ (void)addDoNotBackupAttribute:(NSString *)path    
4、+ (NSString *)md5StringFromString:(NSString *)string 该功能为string 做 MD5 加密;    


#### 4.2.6 YTKBatchRequest 类

功能介绍:批量的网络请求发送，并统一设置它们的回调。    
    
接口协议：    
1、YTKBatchRequestDelegate 该协议提供了批量网络请求成功和失败的响应回调，供委托（delegete）进行实现.    
	
	@optional

	- (void)batchRequestFinished:(YTKBatchRequest *)batchRequest;

	- (void)batchRequestFailed:(YTKBatchRequest *)batchRequest;

	@end


重要成员变量：    
    
1、requestArray，(NSArray *)类型，批量请求数组；    
2、finishedCount，（NSInteger）类型，已完成的请求数目    
3、requestAccessories，（NSMutableArray *）可以hook Request的start和stop的对象数组；    
4、tag,(NSInteger)标签，请求标识。    


重要方法：    

1、- (id)initWithRequestArray:(NSArray *)requestArray; requestArray里面的元素都需要之间或间接的继承YTKRequest;    
2、- (void)startWithCompletionBlockWithSuccess:(void (^)(YTKBatchRequest *batchRequest))success
                                    failure:(void (^)(YTKBatchRequest *batchRequest))failure 设置成功失败的回调并开始请求；    
                                    
3、- (void)clearCompletionBlock，清除block,防止循环引用；    
4、- (void)addAccessory:(id<YTKRequestAccessory>)accessory，添加可以hook Request的start和stop的request对象；    

5、- (void)requestFinished:(YTKRequest *)request （该方法为申明），每一个子请求完成后都调此方法，当所有的子请求都这个类功能统一设置回调的核心方法；    

	- (void)requestFinished:(YTKRequest *)request {
    	_finishedCount++;
    	if (_finishedCount == _requestArray.count) {
        	[self toggleAccessoriesWillStopCallBack];
        	if ([_delegate respondsToSelector:@selector(batchRequestFinished:)]) {
            	[_delegate batchRequestFinished:self];
        	}
        	if (_successCompletionBlock) {
            	_successCompletionBlock(self);
        	}
        	[self clearCompletionBlock];
        	[self toggleAccessoriesDidStopCallBack];
        	[[YTKBatchRequestAgent sharedInstance] removeBatchRequest:self];
    	}
	}

6、- (void)requestFailed:(YTKRequest *)request，请求失败的方法 ，大概思想是：一旦某一个失败了，把还在请求队列中的请求全都暂停，最后再走失败的回调。    

	- (void)requestFailed:(YTKRequest *)request {
    	[self toggleAccessoriesWillStopCallBack];
    // Stop
    	for (YTKRequest *req in _requestArray) {
        	[req stop];
    	}
    	// Callback
    	if ([_delegate respondsToSelector:@selector(batchRequestFailed:)]) {
        	[_delegate batchRequestFailed:self];
    	}
    	if (_failureCompletionBlock) {
        	_failureCompletionBlock(self);
    	}
    	// Clear
    	[self clearCompletionBlock];
    
    	[self toggleAccessoriesDidStopCallBack];
    	[[YTKBatchRequestAgent sharedInstance] removeBatchRequest:self];
	}

##### 4.2.7 YTKBatchRequestAgent 类

功能简介：继承于NSObject,管理YTKBatchRequest 的添加与移除操作。

主要方法：    
1、- (void)addBatchRequest:(YTKBatchRequest *)request 添加YTKBatchRequest对象；    
	
	- (void)addBatchRequest:(YTKBatchRequest *)request {
    	@synchronized(self) {
        	[_requestArray addObject:request];
    	}
	}
	
2、- (void)removeBatchRequest:(YTKBatchRequest *)request，移除YTKBatchRequest对象确保线程同步，加了一把锁；    
	
	- (void)removeBatchRequest:(YTKBatchRequest *)request {
    	@synchronized(self) {
        	[_requestArray removeObject:request];
    	}
	}



##### 4.2.8 YTKChainRequest 类    

功能简介：YTKChainRequest 继承直接于NSObject，负责处理它本身所添加的YTKBaseRequest的顺序请求操作。    


接口协议以及Block：    
1、YTKChainRequestDelegate 该协议提供了请求成功和失败的接口

主要成员变量：    
1、NSMutableArray *requestArray; 链式请求队列（链表）；    
2、NSMutableArray *requestCallbackArray，链式请求回调队列；    
3、NSUInteger nextRequestIndex; 下一个请求索引；    
4、ChainCallback emptyCallback；空回调，不做任何事情，为了保证requestCallbackArray的数目和requestArray而增设了这么一个block，这个block不做任何事；    


主要方法：    
1、- (void)addRequest:(YTKBaseRequest *)request callback:(ChainCallback)callback 添加请求并设置回调，方法内部处理了回调为空的情形；    
	
	- (void)addRequest:(YTKBaseRequest *)request callback:(ChainCallback)callback {
    	[_requestArray addObject:request];
    	if (callback != nil) {
        	[_requestCallbackArray addObject:callback];
    	} else {
        	[_requestCallbackArray addObject:_emptyCallback];
    	}
	}

2、- (void)requestFinished:(YTKBaseRequest *)request:请求完成方法，该方法首先从_requestCallbackArray里面获取request 对于的Callback,让它回调，_requestArray中所有的请求是否已全部完成，如果没有，则继续下一个请求；否则，则回调链式请求结束的方法。    

3、- (void)requestFailed:(YTKBaseRequest *)request，请求失败方法，做一些收尾的工作。    

4、- (BOOL)startNextRequest ,请求开始或者开始下一个请求的方法，衔接两个相邻的请求。    


##### 4.2.9 YTKChainRequestAgent 类

同YTKBatchRequestAgent 基本一样，在此不一一列举。


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

	



