# Cordova源码分析

### 分析点:

## 1.ViewController.m:

### 1.1 xml加载:

具体实现: `loadSetting`方法

```xml
<feature name="YYSContacts"> //插件字典的key
   <param name="ios-package" value="YYSContactsPlugin" /> //插件字典的value
</feature>
```

cordova会从插件字典内根据YYSContacts找到YYSContactsPlugin,作为插件的类名进行初始化操作

### 1.2 webview切换

使用protocol来进行不同webview的创建

```objective-c
@protocol YYSWebViewEngineProtocol <NSObject>

@property (nonatomic, strong, readonly) UIView* engineWebView;

- (id)loadRequest:(NSURLRequest*)request; //加载请求,内部通过此方法进行webview的加载
- (id)loadHTMLString:(NSString*)string baseURL:(NSURL*)baseURL;//加载html
- (void)evaluateJavaScript:(NSString*)javaScriptString completionHandler:(void (^)(id, NSError*))completionHandler;//js与native交互的方法

- (NSURL*)URL;//获取当前网页的url
- (BOOL)canLoadRequest:(NSURLRequest*)request;//判断是否能加载当前请求

- (instancetype)initWithFrame:(CGRect)frame;//设置webview的frame--布局
- (void)updateWithInfo:(NSDictionary*)info; //初始化webview的一些参数

@end
```

具体方法实现:

`newCordovaViewWithFrame`方法

## 2.js<---->native交互

交互分为两种: UIWebView和WKWebView,两者怎么做到统一调用?

### 2.1 UIWebView

UIWebView通过`stringByEvaluatingJavaScriptFromString`进行交互

首先,`cordova.js`向DOM中添加iframe标签,会触发UIWebView的`-(BOOL)webView:(UIWebView*)theWebView shouldStartLoadWithRequest:(NSURLRequest*)request navigationType:(UIWebViewNavigationType)navigationType`代理方法,根据 `gap`标识告诉native有消息要发送过来了  ,native通过调用`cordova.require('cordova/exec').nativeFetchMessages()`,触发js `nativeFetchMessages`的方法,从js获取到要发送过来的信息,回调消息则通过`cordova.require('cordova/exec').nativeCallback('%@',%d,%@,%d, %d)`触发js `nativeCallback`方法,将回调结果告诉js

### 2.2 WKWebView

```objective-c
/*! A class conforming to the WKScriptMessageHandler protocol provides a
 method for receiving messages from JavaScript running in a webpage.
 */
@protocol WKScriptMessageHandler <NSObject>

@required

/*! @abstract Invoked when a script message is received from a webpage.
 @param userContentController The user content controller invoking the
 delegate method.
 @param message The script message received.
 */
- (void)userContentController:(WKUserContentController *)userContentController didReceiveScriptMessage:(WKScriptMessage *)message;

@end
```

`YYSWKWebViewEngine`初始化加入`cordova`这个脚本信息处理器

```objective-c
//Adding a script message handler with name name causes the JavaScript function window.webkit.messageHandlers.name.postMessage(messageBody) to be defined in all frames in all web views that use the user content controller

[userContentController addScriptMessageHandler:weakScriptMessageHandler name:@"cordova"];
```

js端注入  `ios-wkwebview-exec.js`

```javascript
//偷梁换柱
if (window.webkit && window.webkit.messageHandlers && window.webkit.messageHandlers.cordova && window.webkit.messageHandlers.cordova.postMessage) {
    // unregister the old bridge
    cordova.define.remove('cordova/exec');
    // redefine bridge to our new bridge
    cordova.define('cordova/exec', function (require, exports, module) {
        module.exports = execProxy;
    });
}
```

之后可以通过` window.webkit.messageHandlers.cordova.postMessage(command);`向native发送消息

触发`YYSWKWeakScriptMessageHandler`方法:

```objective-c
- (void)userContentController:(WKUserContentController*)userContentController didReceiveScriptMessage:(WKScriptMessage*)message
```

完成传值操作.

回调到js方法:

```objective-c
/* @abstract Evaluates the given JavaScript string.
 @param javaScriptString The JavaScript string to evaluate.
 @param completionHandler A block to invoke when script evaluation completes or fails.
 @discussion The completionHandler is passed the result of the script evaluation or an error.
*/
- (void)evaluateJavaScript:(NSString *)javaScriptString completionHandler:(void (^ _Nullable)(_Nullable id, NSError * _Nullable error))completionHandler;
```



## 3. plugin执行与回调

执行方法:

```objective-c
//@property (nonatomic, readonly) NSArray* arguments; 参数
//@property (nonatomic, readonly) NSString* callbackId; 
//@property (nonatomic, readonly) NSString* className; //类名
//@property (nonatomic, readonly) NSString* methodName; //方法名
- (BOOL)execute:(YYSInvokedUrlCommand*)command
{
    if (![command.className isKindOfClass:[NSString class]]) {
        NSLog(@"ERROR: 类名为非字符串");
        return NO;
    }
    if ((command.className == nil) || (command.methodName == nil)) {
        NSLog(@"ERROR: Classname and/or methodName not found for command.");
        return NO;
    }

    // Fetch an instance of this class 获取实例
    YYSPlugin* obj = [_viewController.commandDelegate getCommandInstance:command.className];

    if (!([obj isKindOfClass:[YYSPlugin class]])) {
        NSLog(@"ERROR: Plugin '%@' not found, or is not a YYSPlugin. Check your plugin mapping in config_loan.xml.", command.className);
        return NO;
    }
    
    BOOL retVal = YES;
    double started = [[NSDate date] timeIntervalSince1970] * 1000.0;
    // Find the proper selector to call. 获取方法名,拼接上:,因为方法为带有参数
    NSString* methodName = [NSString stringWithFormat:@"%@:", command.methodName];
    SEL normalSelector = NSSelectorFromString(methodName);
    //判断插件是否对应的方法
    if ([obj respondsToSelector:normalSelector]) {
        // [obj performSelector:normalSelector withObject:command];
        //执行对应插件的对应方法
        ((void (*)(id, SEL, id))objc_msgSend)(obj, normalSelector, command);
    } else {
        // There's no method to call, so throw an error.
        NSLog(@"ERROR: Method '%@' not defined in Plugin '%@'", methodName, command.className);
        retVal = NO;
    }
    //计算耗时,耗时超过10ms,提示去子线程操作
    double elapsed = [[NSDate date] timeIntervalSince1970] * 1000.0 - started;
    if (elapsed > 10) {
        NSLog(@"THREAD WARNING: ['%@'] took '%f' ms. Plugin should use a background thread.", command.className, elapsed);
    }
    return retVal;
}
```

回调方法:

```objective-c
- (void)sendPluginResult:(YYSPluginResult*)result callbackId:(NSString*)callbackId
{
    YYS_EXEC_LOG(@"Exec(%@): Sending result. Status=%@", callbackId, result.status);
    // This occurs when there is are no win/fail callbacks for the call.
    if ([@"INVALID" isEqualToString:callbackId]) {
        return;
    }
    // This occurs when the callback id is malformed.
    if (![self isValidCallbackId:callbackId]) {
        NSLog(@"Invalid callback id received by sendPluginResult");
        return;
    }
    int status = [result.status intValue];
    BOOL keepCallback = [result.keepCallback boolValue];
    NSString* argumentsAsJSON = [result argumentsAsJSON];
    BOOL debug = NO;
    
#ifdef DEBUG
    debug = YES;
#endif

    NSString* js = [NSString stringWithFormat:@"cordova.require('cordova/exec').nativeCallback('%@',%d,%@,%d, %d)", callbackId, status, argumentsAsJSON, keepCallback, debug];
    [self evalJsHelper:js];
}
```



## 4.runloop优化

Cordova对于插件的执行进行了优化，保证页面的流程度，运用了RunLoop，巧妙的将代码分割为多块分次执行，避免由于插件执行导致主线程阻塞，影响页面绘制，导致掉帧。具体代码如下：

```
// CDVCommandQueue.m

// MAX_EXECUTION_TIME ≈ 1s / 60 / 2
// 计算出绘制一帧时间的一半
static const double MAX_EXECUTION_TIME = .008;

// 判断本次执行时间，如果大于MAX_EXECUTION_TIME，调用performSelector:withObject:afterDelay，结束本次调用
if (([_queue count] > 0) && ([NSDate timeIntervalSinceReferenceDate] - _startExecutionTime > MAX_EXECUTION_TIME)) {
    [self performSelector:@selector(executePending) withObject:nil afterDelay:0];
    return;
}
```

优化策略分析：

- 将队列中的插件分割为很多小块来执行
- 开始执行`executePending`方法时，记录开始时间，每次执行完一个插件方法后，判断本次执行时间是否超过`MAX_EXECUTION_TIME`，如果没有超过，继续执行，如果超过了`MAX_EXECUTION_TIME`，调用`performSelector:withObject:afterDelay`，结束本次调用
- 如果要保证UI流畅，需要满足条件`CPU时间 + GPU时间 <= 1s/60`， 为了给GPU留下足够的时间渲染，要尽量让CPU占用时间小于`1s/60/2`
- Runloop执行的流程如下图所示，系统在收到`kCFRunLoopBeforeWaiting`（线程即将休眠）通知时，会触发一次界面的渲染，也就是在完成`source0`的处理后
- `source0`在这里就是插件的执行代码，在`kCFRunLoopBeforeWaiting`通知之前，如果`source0`执行时间过长就会导致界面没有得到及时的刷新。
- 函数`performSelector:withObject:afterDelay`，会将方法注册到`Timer`，结束`source0`调用，开始渲染界面。界面渲染完成后，`Runloop`开始`sleep`，然后被`timer`唤醒又开始继续处理`source0`。

![img](https://upload-images.jianshu.io/upload_images/2665383-657df331854b7b46.png?imageMogr2/auto-orient/)

## 5. 拦截操作

使用`NSURLProtocol`类进行操作

```objective-c
/*======================================================================
  Begin responsibilities for protocol implementors

  The methods between this set of begin-end markers must be
  implemented in order to create a working protocol.
  ======================================================================*/
  
/*! 
    @method canInitWithRequest:
    @abstract This method determines whether this protocol can handle
    the given request.
    @discussion A concrete subclass should inspect the given request and
    determine whether or not the implementation can perform a load with
    that request. This is an abstract method. Sublasses must provide an
    implementation.
    @param request A request to inspect.
    @result YES if the protocol can handle the given request, NO if not.
*/
//拦截方法里第一个执行的,可以添加标识区分是否要对该请求进行拦截,如果不拦截,返回false,终止
+ (BOOL)canInitWithRequest:(NSURLRequest *)request;

/*! 
    @method canonicalRequestForRequest:
    @abstract This method returns a canonical version of the given
    request.
    @discussion It is up to each concrete protocol implementation to
    define what "canonical" means. However, a protocol should
    guarantee that the same input request always yields the same
    canonical form. Special consideration should be given when
    implementing this method since the canonical form of a request is
    used to look up objects in the URL cache, a process which performs
    equality checks between NSURLRequest objects.
    <p>
    This is an abstract method; sublasses must provide an
    implementation.
    @param request A request to make canonical.
    @result The canonical form of the given request. 
*/
+ (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request;

/*!
    @method requestIsCacheEquivalent:toRequest:
    @abstract Compares two requests for equivalence with regard to caching.
    @discussion Requests are considered euqivalent for cache purposes
    if and only if they would be handled by the same protocol AND that
    protocol declares them equivalent after performing 
    implementation-specific checks.
    @result YES if the two requests are cache-equivalent, NO otherwise.
*/
+ (BOOL)requestIsCacheEquivalent:(NSURLRequest *)a toRequest:(NSURLRequest *)b;

/*! 
    @method startLoading
    @abstract Starts protocol-specific loading of a request. 
    @discussion When this method is called, the protocol implementation
    should start loading a request.
*/
//允许拦截的请求会走到此方法,可以在此处进行本地资源的数据的注入.
- (void)startLoading;

/*! 
    @method stopLoading
    @abstract Stops protocol-specific loading of a request. 
    @discussion When this method is called, the protocol implementation
    should end the work of loading a request. This could be in response
    to a cancel operation, so protocol implementations must be able to
    handle this call while a load is in progress.
*/
- (void)stopLoading;

```

WKWebView不支持拦截操作,但有私有方法可以实现

实现拦截后也会造成post请求体丢失,造成h5的post请求失效



