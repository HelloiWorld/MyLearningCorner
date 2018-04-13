# 读《iOS开发进阶by唐巧》有感

###  写在前面的话
由于这本书已经发行了很久，很多方法已经过时了，所以虽然记录在此，但不保证其可用性。

# 1. 使用CocoaPods做依赖管理
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
	pod update //更新并升级第三方库
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
使用`--no-repo-update`参数禁止更新索引操作：
	
	pod install --no-repo-update
	pod update --no-repo-update
	
### 原理
1. Pods项目最终会编译成一个名为libPods.a的文件，主项目只需要依赖这个.a文件即可。
2. 对于资源文件，CocoaPods提供了一个名为Pods-resources.sh的bash脚本，该脚本在每次项目编译的时候都会执行，将第三方库的各种资源文件复制到目标目录中。
3. CocoaPods通过一个名为Pods,xcconfig的文件在编译时设置所有的依赖和参数。


# 2. 网络封包分析工具Charles
> 具体使用参见[Charles 从入门到精通](http://blog.devtang.com/2015/11/14/charles-introduction/)

关于破解方法自行搜索教程，这里只列出无线调试移动设备步骤：

1. 保持移动设备与电脑所在同一局域网，手动设置HTTP代理为`[local IP address]:8888`，如192.168.100.86:8888
2. 初次连接时，为了能解析HTTPS请求，移动设备要前往chls.pro/ssl安装Charles信任证书（注：iOS 10.3后需进入设置->通用->关于本机->证书信任设置->找到charles proxy custom root certificate，然后启用信任该证书），HTTP无需此步骤
3. 电脑端收到移动设备连接请求并允许，这时再打开App即可

Charles结合Postman工具可以随意任意接口修改参数，开发必备技能


# 3. 界面调试工具Reveal
* 解决了使用纯代码写UI不能即时看到效果的痛苦
* 使用“越狱”设备可以查看其他App的界面布局及排版


# 4. 移动统计工具Flurry
用友盟统计差不多


# 5. 崩溃日志记录工具Crashlytics
被Fabric集成了，同类产品Bugly


# 6. App Store统计工具App Annie
数据那么差，还是不用了🤣


# 7. Xcode插件管理Alcatraz
Xcode8以后就不想再用了


# 8. 其它工具介绍
## 取色工具：数码测色计（DigitalColor Meter）
实用工具，不用再找UI要颜色了，`Shift+Command+C`快速复制RGB值

此外还可以选择下载使用xScope

## 其它图形工具
### ImageOptim（[官网](http://imageoptim.com/))
免费图像压缩工具，减少图片资源占用

PS：此外还有一个图片清理工具[LSUnusedResources](https://github.com/tinymind/LSUnusedResources)，可设定规则清理工程中用不到的图片资源

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