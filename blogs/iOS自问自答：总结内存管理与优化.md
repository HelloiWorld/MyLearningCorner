# 目录
1. ARC下如何避免内存泄露？如何检测？
2. 你是如何做内存优化的？
3. __block你知道多少？在什么时候使用？
4. 关于防止APP崩溃你做了哪些努力？
5. 你是如何做线上Bug定位的？
6. 关于经验和技巧还有什么想说的？


# 1. ARC下如何避免内存泄露？如何检测？ 
* #### 避免：
  - 注意使用block时是否造成循环引用，使用` __weak` 配合 `__strong`关键字打破闭环  

      > **不是所有的block都要避免循环引用**。所谓“循环引用”，指的是双向的强引用（即self强引用block，block也强引用self），单向的强引用则不用担心，系统的某些block api（如`UIView`的block版本写动画）或者第三方库如[MJRefresh](https://github.com/CoderMJLee/MJRefresh)，[SDWebImage](https://github.com/rs/SDWebImage)等未形成闭环均不用考虑循环引用问题。
  - delegate使用`weak`声明比`assign`好，因为使用`weak`其`delegate`成员变量会在持有者销毁时**自动被赋为nil**（[对象回收时Weak指针自动被置为nil的实现原理](http://www.jianshu.com/p/13c4fb1cedea)，典型应用：一句话移除所有通知 `[[NSNotificationCenter defaultCenter] removeObserver:self];`），此时向空对象发消息`objc_msgSend(obj, @selector(methodName:)`判断`obj`为 nil 则`selector`也为 nil 从而直接返回 0(nil) 而不会引起crash
  - 注意`CoreFoundation`对象的使用，使用完成后记得主动调用相应的`CFRelease()`方法
  - 例如`NSTimer`加入到Runloop中，界面消失时记得将定时器销毁，建议使用`NSTimer`的分类(如YYKit的[NSTimer+YYAdd](https://github.com/ibireme/YYKit/tree/master/YYKit/Base/Foundation))，并在`dealloc`中调用`[timer invalidate]`停止定时器

* #### 检测：
> 检测代码中是否存在循环引用问题，可使用 Facebook 开源的一个检测工具[FBRetainCycleDetector](https://github.com/facebook/FBRetainCycleDetector)，这里有两篇很棒的文章翻译并介绍了它的相关用法：  
[[译文]在iOS上自动检测内存泄露](http://ifujun.com/yi-wen-zai-iosshang-zi-dong-jian-ce-nei-cun-xie-lu/)  
[FBMemoryProfiler 基础教程](http://ifujun.com/fbmemoryprofiler-shi-yong-ji-chu-jiao-cheng/)

  - 使用Xcode -> Product -> **Analyze** 分析memory警告，可以发现局部变量忘记release的情况（或者申请了内存却未使用）
  - 在Xcode -> Debug area中点击`Debug Memory Graph`或`Debug View Hierarchy`按钮，在Debug navigator区查看紫色感叹号情况，缺点是每个屏幕都要点击一下
![Debug area](http://upload-images.jianshu.io/upload_images/4483590-fab3f3ed104ca6a6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  - 使用Instruments工具之**Leaks**检测，申请了内存然而没有指向这块内存的指针存在则可以认为是leak了
  - 使用Instruments工具之**Allocations**的mark heap，多次重复操作标记区内存增加应该为0
  - 部分循环引用的地方**Leaks**检测不出来，提供一个思路，结合runtime特性AOP编程，在所有的`dealloc`方法中打印描述信息，若发现退出时不打印，显然是被哪个对象持有了，再仔细排查




# 2. 你是如何做内存优化的？
>  [先弄清楚这里的学问，再来谈 iOS 内存管理与优化（二）](http://www.jianshu.com/p/f95b9bfda4a0) 

  - **Weak-Strong-Dance**防止block和对象间的循环引用（思维延展：多线程下并不一定安全，应对strongSelf进行nil检测 ，[Weak-Strong-Dance真的安全吗？](http://www.jianshu.com/p/737999a30544)）
    > 贴一下[`@weakify`,`@strongify`](http://www.jianshu.com/p/3d6c4416db5e)的宏定义写法
  - 降低内存峰值（reuse，cache，lazy-loading，async就不用多说了）（注：关于属性`strong`定义的`UIView`及其子类，在调用`removeFromSuperView`后并不会立即释放，确定不再使用后可手动置nil）
  - 图片加载方式：带缓存`imageNamed`，不带缓存`contentsOfFile`（适用于大图片） 
    [使用SDWebImage和YYImage下载高分辨率图，导致内存暴增的解决办法](http://www.jianshu.com/p/1c9de8dea3ea) 
     另外加载GIF图，尤其容易导致达到内存峰值从而引起闪退，可采用[FLAnimatedImage解决方案](http://www.jianshu.com/p/10644979f01c)
  - 采用**图片拼合**（多张图片打包整合到一张大图片上显示，cocos2d采用此技术使用OpenGL显示图片，优势：内存使用、载入时间、渲染性能等等）方式，使用时利用`CALayer`的`contentsRect`属性裁剪指定位置的图片，来源[iOS核心动画：寄宿图](http://wiki.jikexueyuan.com/project/ios-core-animation/boarding-figure.html/#db92075a818798745801ef23c3c66ba0)
  - 在for循环中主动添加`@autoreleasepool{}`，保证在每次迭代完成后释放临时变量所占内存（注：当`retainCount=1`且将要release时，打印结果其值不会变为0，原因是为了节省对象引用计数-1的开销，而会在稍后某个时间点或内存不足时释放）
  - 避免滥用单例对象。单例会一直持有资源，合理使用其他替换方案
  - 减少`NSDateFormatter`和`NSCalendar`初始化次数（多次使用模仿单例写法`static`+`dispatch_once`，但更好的方案是使用时间戳如`- (NSDate*)dateFromUnixTimestamp:(NSTimeInterval)timestamp {
return [NSDate dateWithTimeIntervalSince1970:timestamp];
}`）
  - 处理内存警告，释放不再使用的资源

        - (void)didReceiveMemoryWarning {
          [super didReceiveMemoryWarning];//即使没有显示在window上，也不会自动的将self.view释放。注意跟ios6.0之前的区分
          // Add code to clean up any of your own resources that are no longer necessary.
          // 此处做兼容处理需要加上ios6.0的宏开关，保证是在6.0下使用的,6.0以前屏蔽以下代码，否则会在下面使用self.view时自动加载viewDidUnLoad
          if ([[UIDevice currentDevice].systemVersion floatValue] >= 6.0) {
            //需要注意的是self.isViewLoaded是必不可少的，其他方式访问视图会导致它加载，在WWDC视频也忽视这一点。
            if (self.isViewLoaded && !self.view.window)// 是否是正在使用的视图
            {
                // Add code to preserve data stored in the views that might be needed later.
                // Add code to clean up other strong references to the view in the view hierarchy.
                self.view = nil;// 目的是再次进入时能够重新加载调用viewDidLoad函数。
            }
          }
        }


# 3. __block你知道多少？在什么时候使用？
  - 本身不能避免block内部循环引用，可在block内部将 blockObj 置为 nil 的方式避免循环引用
  - 提升了变量的作用域，可以在block内部修改局部变量（通过block进行闭包的变量是`const`的）
  > block不允许修改外部变量的值，这里所说的外部变量的值，指的是栈中指针的内存地址。`__block `所起到的作用就是只要观察到该变量被 block 所持有，就将“外部变量”在栈中的内存地址放到了堆中。进而在block内部也可以修改外部变量的值。
  - 对于变量是指针及数组，只复制了指针，两个指针指向同一个（堆）地址
  - `__block`与`__weak`的区别
  
|  关键字  |  适用模式  |  修饰类型  |  特性  |
|  :---:  |  :----: | :----:  |  :--:  |
|  `__block`  |  ARC、MRC  |  对象、基本数据类型(`int`)  |  可在block块中被重新赋值  |
|  `__weak`  |  ARC  |  对象(`NSString`)  |  对象回收时自动被置为nil  |
> for循环+block块嵌套（常用于多张图片上传及下载）时，可保证block块内部执行完毕后才进入下一次循环，因为实际修改的是同一个地址的内容

##### 一个同步访问数据的栗子
    __block NSDictionary *dict = nil; 
    do { 
        @autoreleasepool {
            NSMutableURLRequest *req = [[NSMutableURLRequest alloc]initWithURL:[NSURL URLWithString:@""]];
            req.timeoutInterval = 10;   // timeoutInterval has no effect

            dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
        
            __block NSURLSessionTask *dataTask = [[NSURLSession sharedSession] dataTaskWithRequest:req completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
                if (!error && data) {
                   dict = [NSJSONSerialization JSONObjectWithData:data options:NSJSONReadingMutableContainers|NSJSONReadingMutableLeaves error:nil];
                } else {
                   NSLog(@"error: %@",[error description]);
                }
                dispatch_semaphore_signal(semaphore);
            }];
            [dataTask resume];
        
            dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
        }
    } while (nil == dict);


# 4. 关于防止APP崩溃你做了哪些努力？
APP发生崩溃常出于以下这些情况：

1. 找不到对应的方法`unrecognized selector sent to instance`
2. 容器越界：数组越界/字典设置空对象等
3. 访问了僵尸对象`EXC_BAD_ACCESS`

解决方案如下：

1. 利用method-swizzling覆盖消息转发链关键方法，具体做法可以参考这篇文章（[iOS 防止应用崩溃解决方案](https://www.jianshu.com/p/d201ad7c9c66)，这个项目也不错[AvoidCrash](https://github.com/frankzhuo/AvoidCrash)），不再赘述
2. 同样是用runtime黑魔法替换方法实现，替换潜在崩溃的方法实现，如增加`NSArray+Safe`分类

        + (void)load {
          static dispatch_once_t onceToken;
          dispatch_once(&onceToken, ^{
              [objc_getClass("__NSArrayI") swizzleSelector:@selector(objectAtIndex:) withSwizzledSelector:@selector(safeObjectAtIndex:)];
          });
        }

        - (id)safeObjectAtIndex:(NSUInteger)index {
          // 数组越界也不会崩，但是开发的时候并不知道数组越界
          if (index > (self.count - 1)) { // 数组越界
              NSLog(@"数组越界了: index = %ld, array = %@", index, self);
              NSAssert(1, @"数组越界了"); // 只有开发的时候才会造成程序崩了
              return nil;
          } else { // 没有越界
              return [self safeObjectAtIndex:index];
          }
        }

3. 在Xcode中启用僵尸调试模式`Zombie Objects`
> 僵尸对象的工作原理是系统即将回收的对象转化为僵尸对象而不彻底回收，在运行时创建一个`_NSZombie_+原类名`的新类，对象的isa指针会被修改指向这个僵尸类，在消息转发机制中`___forwarding___`总是会先检查接收消息的对象所属的类名，一旦发现前缀为`_NSZombie_`，则会特殊处理。

通常发生`EXC_BAD_ACCESS`，需要检查是否有在dealloc中移除通知及KVO（iOS 9以下系统需要手动移除），检查属性关键字是否设置正确，此外还有delegate方法是否判断了事件的响应者等。


虽然有这些解决办法，但是问题依然存在，必然会导致其他问题，crash本来就是帮助开发者找到问题并及时修复，过分的追求减少崩溃率除了保持KPI并不能带来体验上的提升，开发更多的还是要完善容错处理，写出更健壮的代码。



# 5. 你是如何做线上Bug定位的？
> [ iOS 崩溃日志分析](http://blog.csdn.net/my_programe_life/article/details/50686174)  
> [iOS调试之 crash log分析](http://www.jianshu.com/p/12a2402b29c2)  
> [iOS 应用Crash日志分析整理](http://www.jianshu.com/p/45f45590190c)  

- ###### 通过第三方Fabric、Bugly、友盟等SDK上传     
  **缺点**：因为内存占用过大被看门狗杀掉的无法定位出错点，诸如一些野指针问题也没有发现

- ###### 通过Apple后台搜集，可在Xcode -> Window -> Organizer ->Crashes中直接查看定位   
  **缺点**：用户设置不上传诊断信息就看不到日志了 

- ###### [分析iOS Crash文件：符号化iOS Crash文件的3种方法](http://www.cocoachina.com/industry/20140514/8418.html)

|  Require        |  Location  |
|  :----------:  |  ------- | 
| 分析工具symbolicatecrash | 打开终端`find /Applications/Xcode.app -name symbolicatecrash -type f`找到工具所在位置 /Applications/Xcode.app/Contents/SharedFrameworks/DVTFoundation.framework/Versions/A/Resources/symbolicatecrash |
| .dSYM符号文件 | Xcode -> Window -> Organizer -> Archives -> 选中项目 + Download dSYMs…  /右键Show in Finder -> .xcarchive+右键Show in Finder  |
|                            | Xcode -> Products -> .app + 右键Show in Finder |
| crash报告         | Xcode -> Window -> Device -> 选中测试手机 -> View Device Logs  |
|                           |  Finder前往文件夹 -> ~/Library/Logs/CrashReporter/MobileDevice/<DEVICE_NAME>  |
|   **Optional**  |      |
|    .app文件  |  release生成的.ipa文件后缀改为.zip -> 解压 -> Payload目录下的appName.app文件 |
            
###### 终端解析命令： 要求三者在同一文件夹下
 * 先执行`export DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer`，防止出现`Error:"DEVELOPER_DIR" is not defined at ./symbolicatecrash line 60.`
 * `cd`到同一文件夹下，执行`./symbolicatecrash ./*.crash ./*.app.dSYM > symbol.crash`

# 6. 关于经验和技巧还有什么想说的？
都放在github上了 [MyLearningCorner](https://github.com/HelloiWorld/MyLearningCorner)，持续更新ing
