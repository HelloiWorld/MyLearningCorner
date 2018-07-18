# 读唐巧的《iOS开发进阶》有感

###  写在前面的话
由于这本书已经发行了很久，很多方法已经过时了，所以虽然记录在此，但不保证其可用性。<br/>
此外这本书📚争议也相当大，本人不予置评，希望你也能从[唐巧的技术博客](http://blog.devtang.com)中有所获益。


# 2. 使用CocoaPods做依赖管理
## CocoaPods的安装和使用
### CocoaPods的安装
Mac下自带ruby，使用ruby的gem命令下载安装：

	sudo gem install cocoapods
	pod set up

升级gem：
	
	sudo gem update --system

切换gem源：

	gem sources --remove https://rubygems.org/
	gem sources -a http://ruby.taobao.org/
	gem sources -l	//查看列表

### 使用CocoaPods的镜像索引
Podspec文件托管地址：[https://github.com/CocoaPods/Specs](https://github.com/CocoaPods/Specs)

切换gitcafe镜像：

	pod repo remove master
	pod repo add master https://gitcafe.com/akuandev/Specs.git
	// pod repo add master https://git.oschina.net/akuandev/Specs.git
	pod repo update

### 使用CocoaPods

	platform :ios, '8.0' // 指定iOS平台8.0以上系统
	
	pod install //安装依赖库
	pod update //更新并升级第三方库至线上最新版
	pod search [名称] //查找第三方库

### 关于.gitignore
`pod install`初次安装时会将当前各依赖库的版本写入到Podfile.lock文件中，之后再次执行安装也不会更改版本，所以不应该把Podfile.lock文件加入到.gitignore

### 为自己的项目创建podspec文件
	pod spec create your_pod_spec_name

> [《如何编写一个CocoaPods的spec文件》](http://ishalou.com/blog/2012/10/16/how-to-create-a-cocoapods-spec-file/)<br/>
> [《CocoaPods入门》](http://studentdeng.github.io/blog/2013/09/13/cocoapods-tutorial/)

### 使用私有的pods
	pod 'MyCommon', :podspec => 'https://companyname.com/common/myCommon.podspec' 
	
### 不更新podspec
使用`--no-repo-update`参数禁止更新索引操作，只会更新到当前本地库最新版： 
	
	pod install --no-repo-update
	pod update --no-repo-update
	
### 原理
1. Pods项目最终会编译成一个名为libPods.a的文件，主项目只需要依赖这个.a文件即可。
2. 对于资源文件，CocoaPods提供了一个名为Pods-resources.sh的bash脚本，该脚本在每次项目编译的时候都会执行，将第三方库的各种资源文件复制到目标目录中。
3. CocoaPods通过一个名为Pods,xcconfig的文件在编译时设置所有的依赖和参数。


# 3. 网络封包分析工具Charles
> 具体使用参见[Charles 从入门到精通](http://blog.devtang.com/2015/11/14/charles-introduction/)

关于破解方法自行搜索教程，这里只列出无线调试移动设备步骤：

1. 保持移动设备与电脑所在同一局域网，手动设置HTTP代理为`[local IP address]:8888`，如192.168.100.86:8888
2. 初次连接时，为了能解析HTTPS请求，移动设备要前往chls.pro/ssl安装Charles信任证书（注：iOS 10.3后需进入设置->通用->关于本机->证书信任设置->找到charles proxy custom root certificate，然后启用信任该证书），HTTP无需此步骤
3. 电脑端收到移动设备连接请求并允许，这时再打开App即可

Charles结合Postman工具可以随意任意接口修改参数，开发必备技能


# 4. 界面调试工具Reveal
* 解决了使用纯代码写UI不能即时看到效果的痛苦
* 使用“越狱”设备可以查看其他App的界面布局及排版


# 5. 移动统计工具Flurry
用友盟统计差不多


# 6. 崩溃日志记录工具Crashlytics
被Fabric集成了，同类产品Bugly


# 7. App Store统计工具App Annie
数据那么差，还是不用了🤣


# 8. Xcode插件管理Alcatraz
Xcode8以后就不想再用了


# 9. 其它工具介绍
## 取色工具：数码测色计（DigitalColor Meter）
实用工具，不用再找UI要颜色了，`Shift+Command+C`快速复制RGB值

此外还可以选择下载使用xScope

## 其它图形工具
### ImageOptim（[官网](http://imageoptim.com/))
免费图像压缩工具，减少图片资源占用

### 马克鳗（[官网](http://www.getmarkman.com/)）
免费标注工具，给UI的福利

### Dash（[官网](http://kapeli.com/dash)）
API文档查询及代码片段管理工具，支持多编程语言，代码片段管理好以后就不用再去翻工程文件了

### 蒲公英/Fir
内测分发平台，目前已使用fastlane集成

## 命令行工具
### nomad
fastlane的`match`命令估计就是利用了此工具做到证书及描述文件同步的

### xctool（[链接](https://github.com/facebook/xctool)）
xcodebuild的拓展，方便构建、编译和可持续集成

### appledoc
从源代码抽取文档工具，基本不用了，Xcode集成了，使用`Option+Command+/`即可


# 10. 理解内存管理
关于内存管理的一些机制及更原理，推荐去看《Objective-C高级编程：iOS与OS X多线程和内存管理》

个人知识体系里有一些知识点出处在此：
> * 向一个已经被回收的对象发送retainCount消息，它的输出结果应该是不确定的，如果该对象所占的内存被复用了，那么就有可能造成程序异常崩溃。
> * 为什么在这个对象被回收之后，这个不确定的值是1而不是0呢？这是因为当最后一次执行release时，系统知道马上就要回收内存了，就没有必要再将retainCount减1了，因为不管减不减1，该对象都肯定会被回收，而对象被回收后，它的所有的内存区域，包括retainCount值也变得没有意义。不将这个值从1变成0，可以减少一次内存的操作，加速对象的回收。


# 11. 掌握GCD
现在有太多关于它的分享，略


# 12. 使用UIWindow
* UIWindow继承自UIView
* 通常在一个程序中只会有一个UIWindow，它有三个层级`UIWindowLevelNormal` = 0、`UIWindowLevelAlert` = 2000、`UIWindowLevelStatusBar` = 1000，值越大越贴近屏幕。


# 13. 动态下载系统提供的多种中文字体
> [动态下载代码的Demo工程](http://developer.apple.com/library/ios/#samplecode/DownloadFont/Listings/DownloadFont_ViewController_m.html)


# 14. 使用应用内支付
现在内购的分享教程相当多了，快速跳过


# 15. 基于UIWebView的混合编程
iOS8后苹果推出了WKWebView，更流畅，占用更少内存。

### 使用模板引擎渲染HTML界面
* [MGTemplateEngine](http://mattgemmell.com/mgtemplateengine-templates-with-cocoa/)
* [GRMustache](https://github.com/groue/GRMustache)

### OC与JS交互
* [WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge)


# 16. 安全性问题
## 网络安全
### 安全地传输用户密码
> * 原理：黑客使用Charles/Fiddler将自己的电脑设置成代理服务器，就可截取应用的网络请求。如果黑客在公共场所将电脑设置成与该场所名字相同的免费WiFi，一旦有人连接，即可获得发送的数据。
> * 正确做法：事先生成一堆用户加密的公私钥，客户端在登录时，使用公钥将用户密码加密，将密文传输到服务器。服务器使用私钥将密码解密，然后加盐（Salt：在密码学中，是指通过在在密码固定位置插入特定的字符串，让散列后的结果和使用原始密码的散列结果不相符）之后再多次求MD5，之后再和服务器原来存储的用同样方法处理过的密码匹配，如果一致，则登录成功。

现在很多客户端都采用了手机验证码登录，相对安全性增强了许多，但这不是你使用明文传输的理由。

### 防止通讯协议被轻易破解
可使用Protobuf传输数据，使用方法查看博文[iOS-使用Protobuf完成HTTP和Socket通信](https://github.com/HelloiWorld/MyLearningCorner/blob/master/blogs/iOS-使用Protobuf完成HTTP和Socket通信.md)

## 本地文件和数据安全
### 程序文件的安全
通过将JavaScript源码进行混淆和加密，可以防止黑客轻易地阅读和篡改相关的逻辑，也可以防止自己的Web端与Native端的通讯协议泄漏。

### 本地数据安全
对于本地的重要数据，应该加密存储或者将其保存到keychain中，以保证其不被篡改。

## 源代码安全
> IDA([https://www.hex-rays.com/products/ida/](https://www.hex-rays.com/products/ida/))是一个收费的反汇编工具，对于Objective-C代码，它常常可以反汇编到可以方便阅读的程度。应对方法是用一些宏简单混淆类名，关键逻辑用纯C实现。

主推Swift恐怕也是有这里的考虑。


# 17. 基于CoreText的排版引擎
> CoreText是用于处理文字和字体的底层技术。用于复杂的文字排版，如调整文字的字间距必须要使用到它，相比UIWebView它占用内存更少，渲染速度更快，原生交互效果也可以更细腻，不过需要自己处理一些复杂的逻辑。

关于复杂排版如图文混排，可以借鉴作者的实现方案打造专属的排版引擎，具体实现待以后用时查阅。


# 18. 实战技巧
## 申请加急审核
因为之前从未申请过，所以记录一下过程：

1. 访问iTunes Connect网站: [https://itunesconnect.apple.com](https://itunesconnect.apple.com)
2. 单击网站底部的“Contact Us”按钮
3. 在问题1中选择“App Review”
4. 在问题2中选择“Request Expedited Review”
5. 单击“Request an Expedited App Review”按钮即可填写加急审核的申请表

***
* 最好使用英文填写
* 最容易通过理由：严重崩溃Bug(Critical Bug Fix)，详细描述该Bug重现步骤
* 勿滥用

### 测试设备数的限制
* 每年最多可增加100个。
* 急需增加新设备方法：访问[https://developer.apple.com/contact/](https://developer.apple.com/contact/)，单击“Program Benefits“按钮，提交需求。之后会受到处理结果邮件，然后打电话沟通细节。


## 开发技巧
### 收起键盘
1. 重载UIViewController中的`touchesBegan:withEvent:`方法，在其中添加`[self.view endEditing:YES];`方法，可在单击UIViewController的任意地方收起键盘。
2. 直接执行`[[UIApplication sharedApplication] sendAction:@selector(resignFirstResponder) to:nil from:nil forEvent:nil];`
3. 直接执行`[[[UIApplication sharedApplication] keyWindow] endEditing:YES];`

### 设置应用内的系统控件语言
右击Info.plist，Open As -> Source Code，增加

	<key>CFBundleLocalizations</key>
	<array>
			<string>zh_CN</string>
			<string>en</string>
	</array>

也可以这么干：<br/>
Project -> Info -> Localizations增加 Chinese(Simplified)

### 巧用截屏功能
#### 用截屏功能来实现侧滑返回效果
iOS 7后才实现侧滑返回效果，兼容旧版本则需自己动手实现，方法是在push下一个界面前保存一张上一个界面的截屏当做下一界面背景。虽然如今无需这样费力的兼容了，但这个思路很有启发性

#### 用截屏功能来实现半透明效果
1. 半透明看到上一页，采用截屏方式
2. 添加一个新的UIWindow（麻烦的层级关系，也许用来画中画还不错，基本不建议）
3. 采用addChildViewController方式，逻辑多点，但也是我常规想到的方式

### 避免滥用block
block已经看的相当多了，这里不再赘述。

关于block循环引用个人有个快速判断的结论：
* 如果我们用单例持有着对象，通常不用担心在此VC中对单例的持有，只需注意单例对象对block的持有即可，即在使用完后记得调用`self.block = nil;`
* 类方法绝不会产生循环引用问题

### 合并工程文件的冲突
以前与同事协作时，.h和.m有冲突时只用进入所在文件去找`<<<<`或`====`或`>>>>`，改掉错误的位置；<br/>
而如果是.storyboard或者.xib，通常Interface Build就打不开了，这时需要Open As ->  Source Code，然后利用Find(Command+F)去搜索查找；<br/>
如果碰到工程都没法打开的情况，则是去文件所在目录，右击xx.xcodeproj -> 显示包内容 -> 双击project.pbxproj，改掉冲突位置保存即可重新打开工程。

### 忽略编译警告
如果使用了废弃的API，可在Project -> Build Phases -> Compile Sources -> 目标文件的Compiler Flags添加`-w`参数，即可忽略所有烦人（强迫症？）编译警告⚠️，可使用`-Wno-unused-variable`只禁掉未使用变量的编译警告


## Xcode使用技巧
### Xcode快捷键
太多，所以仅熟悉一些自己未掌握而很有必要的组合键：

组合键  | 作用
------------- | -------------
**Cmd + Shift + O**  | 快速查找类，快速跳转到指定类的源代码中，无需再一层层的打开目录
**Ctrl + 6**  | 列出当前文件中所有的方法，此外主要是还能通过输入关键字来过滤
Cmd + Ctrl + ↑/↓ | 在.h和.m文件之间切换
Cmd + Ctrl + ←/→ | 到上/下一次编辑的位置
Cmd + Enter | 切换成Standard Editor
Cmd + Opt + Enter | 切换成Assistant Editor
Cmd + Shift + Y | 切换Console View的显示或隐藏
Cmd + 0 | 隐藏左边导航(Navigator)区
Cmd + Opt + 0 | 隐藏右边工具(Utility)区
Cmd + 1~9 | 切换左边导航子面板
Cmd + Shift + K | Clean
Cmd + . | Stop
ESC | 调出代码补全
Cmd + [ | 指定行向左缩进
Cmd + T | New Tab
Cmd + Shift + [ | 在Tab栏之间切换
--------------------- | -------------------------
Opt | 模拟器双指zoom in/out效果
Cmd + Shift + H | 模拟器的Home键，然而显示模拟器自带了
Cmd + ←/→ | 模拟器切换横竖屏
Cmd + K | 模拟器快捷键，弹出/收起虚拟键盘

### 清除DerivedData
路径：`~/Library/Developer/Xcode/DerivedData`

### 下载Xcode
比如对Xcode9.2(storyboard卡顿，打包iOS8.3以下设备设备图片花屏)不满意，我会再去开发者中心([https://developer.apple.com/downloads/index.action](https://developer.apple.com/downloads/index.action))下载8.3.3版本安装包

### 给模拟器相册增加图片
> 将图片从Finder中拖动到模拟器中，然后用Safari打开，长按图之后选择保存图片

现在不用这么麻烦了，直接拖就自动存到相册里了

### 获得模拟器中的程序数据
我个人图方便使用了Parrot这个应用，Mac App Store里当时减价1块钱买的（1块钱买不了吃亏，买不了上当😂），可以快速进入所有安装了App的模拟器对应App的目录，相当省事。


## ipa文件格式
### 查看ipa的内容
将ipa后缀改成zip，双击解压打开，然后右击.app显示包内容即可。

### 查看ipa中的图片
Xcode自带的图片压缩工具pngcrush地址：/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/usr/bin/

单张图片解压缩：`pngcrush -revert-iphone-optimizations 源文件名 目的文件名`

批量转换脚本：

	#!/bin/bash
	
	#创建解压文件夹
	mkdir OriginImages
	
	for png in `find . -name '*.png'`
	do
		#获得文件名
		name = `basename $png`
		pngcrush -revert-iphone-optimizations $name OriginImages/$name
	done


### 删除未使用的图片资源
这里无需使用作者使用的脚本，推荐[LSUnusedResources](https://github.com/tinymind/LSUnusedResources)，可设定规则清理工程中用不到的图片资源

### 自动生成小尺寸的图片
这里也不用脚本，下载使用App Icon Gear，很方便生成启动图标及多尺寸图片


# 19. Objective-C对象模型
## isa指针
> * 在Objective-C中，每一个对象都有一个名为isa的指针指向该对象的类。
> * 每一个类实际上也是一个对象，也有一个名为isa的指针。
> * 类是元类(metaclass)的实例。所有元类的isa指针都会指向给一个根元类(root metaclass)，根元类本身的isa指针指向自己。
> * 无法在运行时动态地给对象增加成员变量。
> * Category实现原理：通过修改methodLists指针指向的指针的值，动态地为某一个类增加成员方法。

### 系统相关API及应用
#### isa swizzling
* `class_replaceMethod`：替换类方法定义，当需要替换的方法有可能不存在时，会调用class_addMethod新增方法
* `method_exchangeImplementations`：交换两个方法的实现
* `method_setImplementation`：设置一个方法的实现


# 20. Tagged Pointer对象
Tagged Pointer的受益对象是64位系统，现在苹果已不再支持32位系统了，所以都能享受到内存节省和运行效率的提高。Tagged Pointer最后一个bit位是一个特殊的标记，而数据会直接保存在指针本身中。

### Tagged Pointer
1. Tagged Pointer专门用来存储小的对象，例如NSNumber和NSDate。
2. Tagged Pointer指针的值不再是地址，而是真正的值，它只是一个像对象的普通变量。它的内存不存储在堆中，不需要malloc和free。
3. 在内存读取上有着以前3倍的效率，创建时比以前快106倍。

### isa指针
Tagged Pointer是一个伪对象，所以无法直接访问isa成员变量，而应使用方法调用

### 64位下的isa指针优化
64位环境相比32位，对象引用计数的增减操作做了优化和调整。每个对象的Retain操作：

32位环境 | 64位环境
------------- | -------------
1.获得全局的记录引用计数的hash表 | 1.检查isa指针上面的标记位，看引用计数是否保存在isa变量中，如果不是，则使用以前的步骤，否则执行第2步
2.位了线程安全，给该hash表加锁 | 2.检查当前对象是否正在释放，如果是，则不做任何事情
3.查找到目标对象的引用计数值 | 3.增加该对象的引用计数，但是并不马上写回到isa变量中
4.将该引用计数值加1，写会hash表 | 4.检查增加后的引用计数的值是否能够被19位表示，如果不是，则切换成以前的办法，否则执行第5步
5.给该hash表解锁 | 5.进行一个院子的写操作，将isa的值写回

由于去掉了全局的加锁操作，所以引用计数的更改更快了。

### isa的bit位含义
bit位 | 变量名 | 意义
---- | ---- | ----
1 bit | indexed | 0表示普通isa，1表示Tagged Pointer
1 bit | has_assoc | 表示该对象是否有过associated对象，如果没有，在析构释放内存时可以1 bit | has_cxx_dtor | 表示该对象是否有C++或ARC的析构函数，如果没有，在析构释放内存时可以更快
30 bit | shiftcls | 类的指针
9 bit | magic | 其值固定为0xd2，用于在调试时分辨对象是否未完成初始化
1 bit | weakly_referenced | 表示该对象是否有过weak对象，如果没有，在析构释放内存时可以更快
1 bit | deallocating | 表示该对象是否正在析构
1 bit | has\_sidetable_rc | 表示该对象的引用计数值是否大到无法直接在isa中保存
19 bit | extra_rc | 表示该对象超过1的引用计数值，例如，如果该对象的引用计数是6，则extra_rc的值为5


# 21. block对象模型
详阅《Objective-C高级编程：iOS与OS X多线程和内存管理》第2章blocks
