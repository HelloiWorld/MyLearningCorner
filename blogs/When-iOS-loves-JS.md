> 想起了以前在慕课网看到的 @大城小胖 JS混合开发的一个视频，在此整理一下iOS与JS之间的相关知识点。

# JSBinding技术
***
[JSBinding](https://github.com/zynga/jsbindings)技术不是Hybrid技术，它是JavaScript与Native之间的桥接。
![](http://upload-images.jianshu.io/upload_images/4483590-cc83a3f060b5a894.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

JSBinding依赖于JSEngine，而iOS 7首次开放了`JavaScriptCore`的API，使得JSBinding得以打通JS与原生语言的鸿沟。

`JavaScriptCore`的核心类：
- `JavaScriptCore.h`
- `JSContext`
- `JSValue`
- `JSExport`

首先需要导入`JavaScriptCore`头文件
    #import <JavaScriptCore/JavaScriptCore.h>
然后创建运行的上下文
        
    JSContext *context = [[JSContext alloc] init];  // JSContext 是 JS运行环境

增加JS异常处理器

    context.exceptionHandler = ^(JSContext *ctx, JSValue *exception) {
        NSLog(@"%@", exception);
    };


#### 1. Native中调JS

- 执行JS代码：

      JSValue *value = [context evaluateScript:@"1+2"];
      NSLog(@"1 + 2 = %f",[result toDouble]);    //  1 + 2 = 3.000000

- 执行JS函数： 

      NSString *script = @"var function sum(a,b) { return a+b; }";
      [context evaluateScript:script];

      JSValue *sum = context[@"sum"];
      JSValue *result = [sum callWithArguments:@[@1,@2]];
      NSLog(@"sum(1,2) = %f",[result toDouble]);    // sum(1,2) = 3.000000

- 创建JS的值：

      JSValue *intVar = [JSValue valueWithInt32:123 inContext:context];
      context[@"bar"] = intVar;  //不创建全局变量指向值对象
      [context evaluteScript:@"bar++"];

     另一种方式，直接在context中创建全局变量

      [context evaluateScript:@"var bar = 123;"]; 


#### 2. JS通过Block调Native
    context[@"sum"] = ^(int a, int b) {
        return a + b;
    };
    JSValue *result = [context evaluateScript:@"sum(1,2)"];
    NSLog(@"sum(1,2) = %f", [result toDouble]);  // sum(1,2) = 3.000000

可以传递多个参数
   
    context[@"sum"] = ^{
        NSArray *args = [JSContext currentArguments];
        __block double sum = 0;
        [args enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
            sum += [obj toDouble];
        }];
        return sum;
    };
    JSValue *result = [context evaluateScript:@"sum(1,2,3,4,5)"];
    NSLog(@"sum = %f", [result toDouble]); // sum = 15.000000


#### 3.通过JSExport调Native
实例化JSExport类加入到JSContext中，再用JS调用对象

新建一个JSExport类`Point3D`

    #import <JavaScriptCore/JavaScriptCore.h>

    //声明一个协议
    @protocol Point3DExport <JSExport>
    // 将属性暴露给JS
    @property double x;
    @property double y;
    @property double z;
    // 将方法暴露给JS
    - (double)length;
    @end

    @interface Point3D : NSObject <Point3DExport>{
        JSContext *context;
    }

    - (instancetype)initWithContext:(JSContext *)ctx;
    @end

    @implementation Point3D

    @synthesize x,y,z;

    - (instancetype)initWithContext:(JSContext *)ctx {
        if (self = [super init]) {
            context = ctx;
            // 尝试将Point3D类放到上下文中，但实际只是加了一个object，而不是一个类
            context[@"Point3D"] = [Point3D class];
        }
        return self;
    }

    -(double)length {
        return sqrt(self.x * self.x + self.y * self.y + self.z * self.z);
    }

    @end


执行部分

    Point3D *point3D = [[Point3D alloc] initWithContext:context];
    point3D.x = 1;
    point3D.y = 2;
    point3D.z = 3;
    context[@"point3D"] = point3D;
    NSString *script = @"point3D.x=0;point3D.y=4;point3D.length()";
    JSValue *result = [context evaluateScript: script];
    NSLog(@"Result of %@ is %f", script, [result toDouble]);  // Result of point3D.x=0;point3D.y=4;point3D.length() is 5.000000

#### 4.直接加载JS文件
    void loadScriptContext(JSContext *ctx, NSString *fileName) {
        NSString *filePath = [NSString stringWithFormat:@"%@/JS/%@",[[NSBundle mainBundle] resourcePath], fileName];
        NSString *script = [NSString stringWithContentsOfFile:filePath encoding:NSUTF8StringEncoding error:nil];
        [ctx evaluateScript:script];
    }
可以借用3中的方式，模拟控制台的功能，然后利用`[JSContext currentArguments]`获取所有的参数打印。

### 关于JS与OC之间类型转换
![OC与JS的类型转换](http://upload-images.jianshu.io/upload_images/4483590-190bcd985eec0cc2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 关于JS与OC之间Retain cycle的处理
使用`JSManagedValue`包装JS对象，系统会正确处理垃圾回收及内存管理。

    [JSManagedValue managedValueWithValue:value];

### 关于JS与OC之间多线程的情况
同一个`JSVirtualMachine`相当于处于同一个线程下，不用担心锁及并发问题。

    // 初始化虚拟机，类似于创建了一个新线程
    JSVirtualMachine *jsvm = [[JSVirtualMachine alloc] init];
    JSContext *ctx = [[JSContext alloc] initWithVirtualMachine:jsvm];


# Hybrid
***
即混合开发，由Native通过`JSBridge`等方法提供统一的API，然后用Html5+JS来写实际的逻辑，调用API，这种模式下，由于Android，iOS的API一般有一致性，而且最终的页面也是在webView中显示，所以有跨平台效果。比较有名的框架有PhoneGap([Cordova](http://www.jianshu.com/p/fb0149025869))，AppCan等。
Hybrid 是Web技术与Native之间的桥梁！
![](http://upload-images.jianshu.io/upload_images/4483590-bf62f06914fbae90.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 1. 原生代码调用网页中的 JavaScript 函数
假设我们的网页中有如下代码

    [script type="text/javascript"]

    function myFunc() {
        return "Text from web"
    }

    [/script]

原生代码可以用如下方式调用`myFunc()`

    NSString * result = [self.webView stringByEvaluatingJavaScriptFromString:@"myFunc()"];
    NSLog(@"%@",result);
打印结果为 Text from web

#### 2. 网页中的 JavaScript 调用系统的原生代码
这一步比上边的要复杂一些，iOS 不像 Android 可以直接给网页中的 JavaScript 函数注入一个原生代码的接口。这里我们会用一个比较曲折的方式来实现。

假设 Objective-C 的类里有一个方法

    -(void)nativeFunction:(NSString*)args {
    }
JavaScript 里我们用下边的方法来最终调用到上边这个方法

    window.JSBridge.callFunction("callNativeFunction", "some data");

在我们的页面里，是没有`JSBridge.callFunction`存在的，这一步我们要在原生代码端注入。

在 webView 的 delegate 的`- (void)webViewDidFinishLoad:(UIWebView *)webView`里我们用下边的方式注入 JavaScript

    NSString *js = @"(function() {
        window.JSBridge = {};
        window.JSBridge.callFunction = function(functionName, args) {
            var url = \"bridge-js://invoke?\";
            var callInfo = {};
            callInfo.functionname = functionName;
            if (args) {
                callInfo.args = args;
            }
            url += JSON.stringify(callInfo);
            var rootElm = document.documentElement;
            var iFrame = document_createElement_x_x_x(\"IFRAME\");
            iFrame.setAttribute(\"src\",url);
            rootElm.a(iFrame);
            iFrame.parentNode.removeChild(iFrame);
        };
        return true;
    })();";

    [webView stringByEvaluatingJavaScriptFromString:js];
简单解释一下，首先我们在 window 里创建一个叫`JSBridge`的对象，然后在里边定义一个方法 `callFunction`，这个方法的作用是把两个参数打包为 JSON 字符串，然后附带到我们自定义的`URL bridge-js://invoke?`后边，最后用`IFRAME`的方式来加载这个 URL

这么做的原因是，当加载`IFRAME`的时候，就会调用 webView 的 delegate 的`- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType`方法，其中 request 就是我们刚才自定义的那个 URL，在这个方法里我们做如下处理

    NSURL *url = [request URL];
    NSString *urlStr = url.absoluteString;
    return [self processURL:urlStr];
processURL 函数如下

    - (BOOL)processURL:(NSString *) url {
        NSString *urlStr = [NSString stringWithString:url];
        NSString *protocolPrefix = @"bridge-js://invoke?";
        if ([[urlStr lowercaseString] hasPrefix:protocolPrefix]) {
            urlStr = [urlStr substringFromIndex:protocolPrefix.length];
            urlStr = [urlStr stringByReplacingPercentEscapesUsingEncoding:NSUTF8StringEncoding];
            NSError *jsonError;
            NSDictionary *callInfo = [NSJSONSerialization JSONObjectWithData:[urlStr dataUsingEncoding:NSUTF8StringEncoding] options:kNilOptions error:&jsonError];
            NSString *functionName = [callInfo objectForKey:@"functionname"];
            NSString * args = [callInfo objectForKey:@"args"];
            if ([functionName isEqualToString:@"callNativeFunction"]) {
                [self nativeFunction:args];
            }
            return NO;
        }
        return YES;
    }

从`bridge-js://invoke?`这个自定义的 URL 里边把附带在后边 JSON 字符串解析出来，然后判断 `function name` key的值如果是`callNativeFunction`那么就去调用原生方法`nativeFunction`, 如果需要实现更多的方法调用，只要添加这个映射关系就行了。

至此，JavaScript 和 Objective-C 代码的双向调用就都实现了。

### 思路
更好的一个JSBridge的使用方案：[WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge)


# React Native
***
Facebook发起的开源的一套新的APP开发方案，使用JS+部分原生语法来实现功能，底层会把React转换为原生API，相比Weex，iOS与Android不共用同一套代码。
基于React，宣称“Learn once, write anywhere”
抽空准备学习中...


# Weex
***
Weex来自阿里系，最底层的原理是和React-Native相同的，就是将JS代码渲染成原生组件。
基于Vue，宣称"Write once, run anywhere"
犹豫到底学哪个😂
学习推荐作者：@[一缕殇流化隐半边冰霜](http://www.jianshu.com/u/12201cdd5d7a)
