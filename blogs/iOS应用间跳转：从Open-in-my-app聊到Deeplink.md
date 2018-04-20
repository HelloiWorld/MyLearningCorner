就我个人所知，iOS中存在3种方式可以打开（唤醒）其它手机App（除开系统级应用），分别是：
- 第三方登录、分享、支付、导航等
- Open in my app
- Deeplink

三种方式原理一样，均是注册自定义`URL Schemes`，并处理URL请求。
![URL schemes.jpeg](http://upload-images.jianshu.io/upload_images/4483590-fce645ae8b0ede32.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


# 第三方
***
1. 使用第三方用户登录，如微信，QQ，微博登录，授权后返回用户名和密码
2. 内容分享，跳转到分享App的对应页面，如分享给微信好友、分享给微信朋友圈、分享到微博
3. 第三方支付，跳转到第三方支付App，如支付宝支付，微信支付
4. 显示位置、地图导航，跳转到地图应用，如高德地图，百度地图等
5. 应用程序推广，跳转到iTunes并显示指定App下载页，或使用Safari打开指定网页

1~4 第三方平台均提供了相应SDK，具体不再阐述，5实际只需要一个网址，调用`[[UIApplication sharedApplication] openURL:url]`方法即可。
> 在iOS9中，如果使用`canOpenURL:`方法，该方法所涉及到的`URL Schemes`必须在"Info.plist"中将它们列为白名单，否则不能使用。key叫做`LSApplicationQueriesSchemes`，键值内容是对应应用程序的`URL Schemes`。


# Open in my app
***
iOS有个不常使用的功能，叫Open in my app，即在我的app中打开，此功能允许文件在其他app中打开。

常见如邮件的附件，**轻触**苹果会默认使用`QLPreviewController`打开预览界面，而**长按**这时会弹出共享列表菜单，此菜单会列出所有添加了该类型文件的应用，它允许此文件在添加了对应`Document Type`支持的应用中打开，如下图所示：
![Open in my app](http://upload-images.jianshu.io/upload_images/4483590-8163dd03ba2bf443.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 实现步骤
#### 1. 修改Info.plist文件
**1）在plist文件中添加URLTypes字段，指定程序的对外接口：**
![Info.plist](http://upload-images.jianshu.io/upload_images/4483590-ce6e3dda85936e04.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
由于我之前已经集成了社会化分享，这一步就直接跳过。

**2）添加一个Documents Type字段，该字段指定与程序关联的文件类型，详情参考[System-Declared Uniform Type Identifiers](https://developer.apple.com/library/ios/documentation/Miscellaneous/Reference/UTIRef/Articles/System-DeclaredUniformTypeIdentifiers.html#//apple_ref/doc/uid/TP40009259-SW1)**
> `CFBundleTypeExtensions`指定文件类型，例如`pdf`，`doc`，`xls`，`ppt`，`txt`等。
`CFBundleTypeIconFiles`指定用`UIActionSheet`向用户提示打开应用时显示的图标。
`DocumentTypeName`可以自定，对应文件类型名。
`Document Content Type UTIs`指定官方指定的文件类型，UTIs 即 Uniform Type Identifiers。

- **PDF文件**
![.pdf的配置](http://upload-images.jianshu.io/upload_images/4483590-6f954b7bf5293a04.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- **Doc文件**
![.doc(s)的配置](http://upload-images.jianshu.io/upload_images/4483590-203a43f173ee0990.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- **Excel文件**
![.xls(x)的配置](http://upload-images.jianshu.io/upload_images/4483590-bf9ad1e0e04990d3.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- **PPT文件**
![.ppt的配置](http://upload-images.jianshu.io/upload_images/4483590-73bbe0fc98dea16d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
> 注意：预支持PPT需要 额外配置增加`public.data`字段，被这个坑了好久!

- **TXT（或RTF）**
![.txt的配置](http://upload-images.jianshu.io/upload_images/4483590-1aa6ddb6146fc8dd.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 2. 在 AppDelegate.m 中添加外部调用代码
    #import <QuickLook/QuickLook.h>

    @interface AppDelegate ()  <QLPreviewControllerDataSource>
    @property (nonatomic,strong) NSURL *openUrl; 
    @end

**注意：iOS 9之后请使用这个方法：**
`- (BOOL)application:(UIApplication *)app openURL:(NSURL *)url options:(NSDictionary<NSString*, id> *)options;`

    -(BOOL)application:(UIApplication *)application openURL:(NSURL *)url sourceApplication:(NSString *)sourceApplication annotation:(id)annotation {
      if (url && [url isFileURL]) {
        self.openUrl=url;       

        QLPreviewController* previewController = [[QLPreviewController alloc] init];
        [self.window.rootViewController presentViewController:previewController animated:YES completion:nil];    

        dispatch_async(dispatch_get_global_queue(0, 0), ^{      
            NSData *data = [NSData dataWithContentsOfURL:url];
            dispatch_async(dispatch_get_main_queue(), ^{
                NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
                NSString *documentsPath=[[paths objectAtIndex:0] stringByAppendingPathComponent:@"downloadFile"];      
                BOOL dirHas;
                if (![[NSFileManager defaultManager] fileExistsAtPath:dirPath isDirectory:&dirHas] ) {
                    [[NSFileManager defaultManager] createDirectoryAtPath:dirPath withIntermediateDirectories:NO attributes:nil error:nil];
                }  
                NSString *filePath = [documentsPath stringByAppendingPathComponent:[[url relativePath] lastPathComponent]];//Add the file name
                [data writeToFile:filePath atomically:YES];
                //写入成功后再读取刷新数据，避免跳转界面时等待太久
                previewController.dataSource = self;
                [previewController reloadData];
            });
        });
        return YES;
      }
      return NO;
    } 
    
    //使用QLPreviewController预览
    - (NSInteger) numberOfPreviewItemsInPreviewController: (QLPreviewController *) controller {
        return 1;
    }

    - (id <QLPreviewItem>)previewController: (QLPreviewController*)controller previewItemAtIndex:(NSInteger)index {
    //    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
    //    NSString *docDir = [[paths objectAtIndex:0] stringByAppendingPathComponent:@"Inbox"];
        NSString *filePath=[getDocumentPath() stringByAppendingPathComponent:[[self.openUrl relativePath] lastPathComponent]];
         if ([[filePath pathExtension] isEqualToString:@"txt"]) {
            //处理txt格式内容显示有乱码的情况
            NSData *fileData = [NSData dataWithContentsOfFile:filePath];
            //判断是UNICODE编码
            NSString *isUNICODE = [[NSString alloc] initWithData:fileData encoding:NSUTF8StringEncoding];
            //还是ANSI编码（-2147483623，-2147482591，-2147482062，-2147481296）encoding 任选一个就可以了
            NSString *isANSI = [[NSString alloc] initWithData:fileDataencoding:-2147483623];
            if (isUNICODE) {
                NSString *retStr = [[NSString alloc]initWithCString:[isUNICODE UTF8String] encoding:NSUTF8StringEncoding];
                NSData *data = [retStr dataUsingEncoding:NSUTF16StringEncoding];
                [data writeToFile:filePath atomically:YES];
            } else if (isANSI) {
                NSData *data = [isANSI dataUsingEncoding:NSUTF16StringEncoding];
                [data writeToFile:filePath atomically:YES];
            }
        }
        NSURL *fileUrl = [NSURL fileURLWithPath:filePath];
       return fileUrl;
    }

**至于为什么要下载?**
我经过测试发现文件都是存在沙盒`Documents/Inbox/`目录下的，获取到的文件路径例如：`file:///private/var/mobile/Containers/Data/Application/A9D2A924-A1F8-4915-B31D-57127ED987BE/Documents/Inbox/工作日报_0113.xlsx`，然而却怎么也打不开，尝试拼接取文件路径也不成功，于是重新下载了一份放到我指定的目录下，也可以直接使用`copyItemAtPath: toPath: error:`方法拷贝文件到指定的路径下。

### 更多支持文件类型（如MP3,AIFF,WAV, etc.）
stack overflow给出了各个类型所需要添加的plist字段：
[http://stackoverflow.com/questions/9266079/why-is-my-ios-app-not-showing-up-in-other-apps-open-in-dialog](http://stackoverflow.com/questions/9266079/why-is-my-ios-app-not-showing-up-in-other-apps-open-in-dialog)


# Deeplink（深度链接）
***
Deeplink，简单来说就是给定一个链接，即可跳转到**已安装应用**中的某一个页面的技术。

通过Deep Link，App可以通过搜索引擎进行导流。可以通过 SEO 增加访问量从而提高 app 下载量和开启率，可以实现Web与App间切换保留上下文信息。


最初听到Deeplink是在[魔窗](http://www.magicwindow.cn)的宣讲会，那时一脸懵，回来仔细了解了下，相比推送跳转的方式（现在很多用户都是直接关闭推送的），通过分享邀请和短信唤醒，对运营拉新、拉活、留存、转化，提供了更多帮助。

###### 技术要求：
-  APP要想被其他APP直接打开，自身得支持，让自己具备被人打开的能力。（URL Schemes）
-  APP要想打开其他的APP，自身也得支持。（判断设备是否安装、各种跳转的处理）
> iOS 9以上的用户，可以通过`Universal Links`通用链接无缝的重定向到一个App应用，而不需要通过Safari打开跳转。
如果用户没有安装这个app，则会在Safari中打开这个链接指向的网页。
唤醒方法为:`- (BOOL)application:(UIApplication *)application continueUserActivity:(NSUserActivity *)userActivity restorationHandler:(void (^)(NSArray * _Nullable))restorationHandler;`

栗子🌰(关于Universal Links的具体配置建议查看[官方文档](https://developer.apple.com/library/content/documentation/General/Conceptual/AppSearch/UniversalLinks.html#//apple_ref/doc/uid/TP40016308-CH12-SW2)  苹果有个网址可以测试apple-app-site-association是否有效[测试地址](https://search.developer.apple.com/appsearch-validation-tool/))

    - (BOOL)application:(UIApplication *)application continueUserActivity:(NSUserActivity *)userActivity restorationHandler:(void (^)(NSArray *restorableObjects))restorationHandler{
      if ([userActivity.activityType isEqualToString:NSUserActivityTypeBrowsingWeb]) {
        NSURL *webUrl = userActivity.webpageURL;
        if ([webUrl.host isEqualToString:@"com.example.ios"]) {
            //打开对应页面
    //            NSString *paramStr = [[webUrl query] substringFromIndex:[[webUrl query] rangeOfString:@"?"].location+1];
            NSDictionary *paramDic = [[webUrl absoluteString] getURLParameters];
            NSLog(@"paramDic: %@",paramDic);
            //跳转本地ViewController
        } else {
            //不能识别，safari打开
            [[UIApplication sharedApplication] openURL:webUrl];
        }
      }
      return YES;
    }

    //  NSString+UrlEncoding.h
    /**
     *  截取URL中的参数
     *
     *  @return NSDictionary parameters
     */
    - (NSDictionary *)getURLParameters {
    
    // 查找参数
    NSRange range = [self rangeOfString:@"?"];
    if (range.location == NSNotFound) {
        return nil;
    }
    
    NSMutableDictionary *params = [NSMutableDictionary dictionary];
    
    // 截取参数
    NSString *parametersString = [self substringFromIndex:range.location + 1];
    
    // 判断参数是单个参数还是多个参数
    if ([parametersString containsString:@"&"]) {
        
        // 多个参数，分割参数
        NSArray *urlComponents = [parametersString componentsSeparatedByString:@"&"];
        
        for (NSString *keyValuePair in urlComponents) {
            // 生成Key/Value
            NSArray *pairComponents = [keyValuePair componentsSeparatedByString:@"="];
            NSString *key = [pairComponents.firstObject stringByRemovingPercentEncoding];
            NSString *value = [pairComponents.lastObject stringByRemovingPercentEncoding];
            
            // Key不能为nil
            if (key == nil || value == nil) {
                continue;
            }
            
            id existValue = [params valueForKey:key];
            
            if (existValue != nil) {
                
                // 已存在的值，生成数组
                if ([existValue isKindOfClass:[NSArray class]]) {
                    // 已存在的值生成数组
                    NSMutableArray *items = [NSMutableArray arrayWithArray:existValue];
                    [items addObject:value];
                    
                    [params setValue:items forKey:key];
                } else {
                    // 非数组
                    [params setValue:@[existValue, value] forKey:key];
                }
                
            } else {
                // 设置值
                [params setValue:value forKey:key];
            }
        }
    } else {
        // 单个参数
        
        // 生成Key/Value
        NSArray *pairComponents = [parametersString componentsSeparatedByString:@"="];
        
        // 只有一个参数，没有值
        if (pairComponents.count == 1) {
            return nil;
        }
        
        // 分隔值
        NSString *key = [pairComponents.firstObject stringByRemovingPercentEncoding];
        NSString *value = [pairComponents.lastObject stringByRemovingPercentEncoding];
        
        // Key不能为nil
        if (key == nil || value == nil) {
            return nil;
        }
        
        // 设置值
        [params setValue:value forKey:key];
    }
    
    return [NSDictionary dictionaryWithDictionary:params];
    }
 
测试方法：在短信或备忘录中输入相应域名，若能跳转到相应App即植入成功。直接在Safari中输入链接是无效的，必须从一处跳入才可以（比如上一级网页）。


### Deferred Deeplink（延迟深度链接）
Deeplink只针对手机中已经安装过App的用户才有用。而升级版本的**Deferred Deeplink**却可以解决这个问题：
> **Deferred Deeplink 可以先判断用户是否已经安装了App应用，如果没有则先引导至App应用商店中下载App， 在用户安装App后跳转到指定App页面 Deeplink 中。**

##### Deferred Deeplink的应用场景：
- 追踪广告效果
- 追踪用户推荐/邀请链接
- 在 app 内保持网页浏览的上下文，如登录信息，购物车等

App分享邀请好友，好友通过链接（只有通过此链接跳转到App Store下载App才算有效）安装App之后双方获得奖励，省去了过去注册输入邀请码这一步。
