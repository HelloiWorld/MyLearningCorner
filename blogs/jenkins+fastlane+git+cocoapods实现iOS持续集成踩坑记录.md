# 前提
本项目在配置jenkins前已配置安装fastlane并自动上传蒲公英，关于fastlane的使用不在本文讨论范围之内。

#安装Jenkins
jenkins有几种方式安装，一种是去[官网](https://jenkins.io/download/)下载dmg安装包(还可以下载.war文件，通过执行命令`java -jar jenkins.war`安装)，这也是我最先选择的方式，然而此种方式安装确有一些很明显的坑
1. 输入初始密码需要前往`/Users/Shared/Jenkins/Home/`这个目录下，非Jenkins用户需要给`/secrets/`增加读权限，然后找到`initialAdminPassword`文件，打开复制出密码，在初次登入[http://localhost:8080](http://localhost:8080)时使用
2. 上面的还只是小问题，最大的问题来了。本人正确配置项目后，通过git拉取代码时总是出现timeout超时问题，查询了一堆资料，比如增加超时时间，只拷贝最近一次的代码
![git fetch --tags --progress time out.png](http://upload-images.jianshu.io/upload_images/4483590-e185b9a7eb8247d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
然而，并没有什么luan用，通过40+次构建失败，让我认识到此法不通，于是卸载重装
安装包卸载路径： `/Library/Application Support/Jenkins/Uninstall.command`
brew方式卸载方法：`brew uninstall jenkins`

推荐的安装方式是通过**homebrew**安装，此方案安装后以上问题不再出现，其安装目录为/Users/当前用户/.jenkins，隐藏文件需使用命令`command+shift+.`显示
安装命令: `brew install jenkins`
> 你可能需要了解这些[gem、brew、rvm、bundle的相关介绍](http://blog.csdn.net/lizitao/article/details/52575334)

![](http://upload-images.jianshu.io/upload_images/4483590-5b19ac5a3e34b950.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如果出现以上问题，说明brew版本不匹配，可以执行`brew --version`命令会自动更新到最新版本
![](http://upload-images.jianshu.io/upload_images/4483590-10603c1723bbb1c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


顺利的执行完`brew install jenkins`，结果发现安装的是2.68版本，根据提示命令更新到新版2.101

#启动Jenkins
执行命令`jenkins`，浏览器打开http://localhost:8080 web页面，粘贴所获的的初始密码
![initialAdminPassword.png](http://upload-images.jianshu.io/upload_images/4483590-83b95c2e8f3cc89b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 插件plugin
我只安装了社区推荐的插件(安装时建议翻墙)，可选插件[Xcode integration](https://wiki.jenkins-ci.org/display/JENKINS/Xcode+Plugin)
与其他文章不同的是，[Keychains and Provisioning Profiles Management](https://plugins.jenkins.io/kpp-management-plugin)这个插件我并没有选择安装，原因是这些东西我已经通过fastlane在脚本配置里，无需再另行上传证书及相关设置
如果插件安装失败，可前往http://localhost:8080/pluginManager/available更新插件

### 系统设置
##### GitHub设置
如果项目是GitHub私有项目，或是使用github轮询，需要添加GitHub Server及许可证
![GitHub Server](http://upload-images.jianshu.io/upload_images/4483590-9e163406fd6e3d2d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

github需要授权才能使用相关api，access tokens可在 Settings->Developer settings->Personal access tokens里管理
![Personal access tokens.png](http://upload-images.jianshu.io/upload_images/4483590-8f598bedcc8633a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##### 配置邮件地址
![配置系统管理员邮件地址](http://upload-images.jianshu.io/upload_images/4483590-3e0d79da33d59347.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这个警告⚠️我是保留的，如果修改为本地127.0.0.1会提示反向代理设置有误
**由于本机没有公网ip，所以导致git轮询的方式会不可用，在此记录一下**

![Extended E-mail Notification](http://upload-images.jianshu.io/upload_images/4483590-5f445e3b0e02bacf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
###### Default Content: 

(本邮件是 Jenkins 服务端构建完毕后自动发送，请勿回复)<br/><hr/>

项目名称：$PROJECT_NAME<br/>
Git版本号：$GIT_REVISION<br/>
触发原因：$CAUSE<br/>
构建编号：# $BUILD_NUMBER<br/>
构建状态：$BUILD_STATUS<br/>
构建日志地址：<a href="${BUILD_URL}console">${BUILD_URL}console</a><br/>
构建地址：<a href="$BUILD_URL">$BUILD_URL</a><br/>
本次构建变化：$CHANGES_SINCE_LAST_SUCCESS<br/>

![send email success.png](http://upload-images.jianshu.io/upload_images/4483590-6a48ad076fae766c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果出现***Failed to send out email javax.mail.AuthenticationFailedException: 535 Error: authentication failed, system busy***错误，在确保邮箱已开启SMTP服务情况下，请保证**Extended E-mail Notification**与**邮件通知**两者的配置是一致的，并检查密码。
如果提示501错误，这是因为你的系统管理员地址与授权的账号地址不是同一个，需要保持一致。

# 添加项目地址
私有项目如果报权限问题，最好在本机生成SSH Key，这时你需要去`~/.ssh/id_rsa`文件里拿取私钥信息，然后加到Global Credentials中即可。

# 构建Job
通过github构建项并添加源码管理
![](http://upload-images.jianshu.io/upload_images/4483590-2392ec3fab45d483.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
当然你也可以选择Build periodically或Poll SCM方式
- **Build periodically**：周期构建，它不关心源码是否变化
- **Poll SCM**：定时检查源码变更，如果有更新就checkout最新code下来，然后执行构建动作
![](http://upload-images.jianshu.io/upload_images/4483590-5209048b7088bccd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


* * * * *
第一个*表示分钟，取值0~59
第二个*表示小时，取值0~23
第三个*表示一个月的第几天，取值1~31
第四个*表示第几月，取值1~12
第五个*表示一周中的第几天，取值0~7，其中0和7代表的都是周日


##### fastlane脚本
>  fastlane项目地址：[https://github.com/fastlane/fastlane](https://github.com/fastlane/fastlane)
fastlane相关文档：[docs.fastlane.tools](https://docs.fastlane.tools/)

你可以直接使用
```
IPANAME="jinkens-myapp"
fastlane gym --export_method ad-hoc --output_name ${IPANAME}
```
来指定打包方式（[蒲公英文档](https://www.pgyer.com/doc/view/jenkins_ios)），也可以手动配置，以下是我fastlane关于pgy这个action的代码(我的fastlane已通过`match`自动配置证书，关于这个命令的使用可以查看[fastlane match命令](https://docs.fastlane.tools/actions/match/))

desc "Submit a new Beta Build to Pgy"
lane :pgy do
match(type: "adhoc", app_identifier: "com.xxxxxx.ios", readonly: true)
gym(
scheme: "xxx",
silent: true,  # 是否隐藏打包时不需要的信息
configuration: 'Debug', # 指定打包时的配置项，默认为Release 
export_method: "ad-hoc", # 指定导出.ipa时使用的方法，可用选项：app-store, ad-hoc, package, enterprise, development, developer-id
export_options: {
provisioningProfiles: {
"com.xxxxxx.ios" => "match AdHoc com.xxxxxx.ios"
}
}
)
pgyer(api_key: "xxxxxxxxxxxxxxxx", user_key: "xxxxxxxxxxxxxxxxxxx")
end

##### 构建后压缩存档
![Archive the artifacts](http://upload-images.jianshu.io/upload_images/4483590-3bf2f34d241e11f6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##### 构建后打包并上传到蒲公英
除了在fastlane安装蒲公英插件的方式，还可以在jenkins中选择安装`Upload to pgyer`插件
![](http://upload-images.jianshu.io/upload_images/4483590-8a287d4918621b0d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![Upload to pgyer](http://upload-images.jianshu.io/upload_images/4483590-3dc660a48a55da59.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##### 构建后上传fir.im
需要先安装`fir-plugin`插件
![Upload to fir.im](http://upload-images.jianshu.io/upload_images/4483590-74af1fbda1a3f42a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


##### 构建后添加邮件通知
![Editable Email Notification](http://upload-images.jianshu.io/upload_images/4483590-7e66352b70083c3d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
点开高级设置`Advanced settings...`，根据构建状态发信
![Advanced settings](http://upload-images.jianshu.io/upload_images/4483590-ec469e41ed1f173e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


# 执行Job
点击“立即构建”，未加入 fastlane相关控制台输出如下：
![first_log.png](http://upload-images.jianshu.io/upload_images/4483590-41851192dd8e29ed.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### Share Scheme
以为这就完了吗？我还真以为这样就搞定了-_-
执行`fastlane pgy`构建失败！CI下如果使用了workspace，你需要显式的指定一个scheme：

Couldn't find specified scheme 'xxx'.
Multiple schemes found but you haven't specified one.
Since this is a CI, please pass one using the `scheme` option
Couldn't find any schemes in this project, make sure that the scheme is shared if you are using a workspace
Open Xcode, click on `Manage Schemes` and check the `Shared` box for the schemes you want to use
Afterwards make sure to commit the changes into version control

以前在本地运行可没这情况，赶紧按其解决办法在`Manage Schemes`中将Shared勾选☑️
![Manage Schemes.png](http://upload-images.jianshu.io/upload_images/4483590-9de482fe03d7fb52.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
别忘了将相关设置上传，以下是我参考[[iOS] gitignore 忽略上传pods/cocoaPods 文件](http://www.jianshu.com/p/4ed175f13e97)修改过的.gitignore文件内容

# Xcode
.DS_Store
*/build/*
*.pbxuser
!default.pbxuser
*.mode1v3
!default.mode1v3
*.mode2v3
!default.mode2v3
*.perspectivev3
!default.perspectivev3
xcuserdata
!xcshareddata
profile
*.moved-aside
DerivedData
.idea/
*.hmap
*.xccheckout
*.xcworkspace
*.xcuserstate
!default.xcworkspace

#CocoaPods
Pods
!Podfile
!Podfile.lock

#Fastlane
!fastlane
!Gemfile
!Gemfile.lock
*.ipa
*.app.dSYM.zip


# 最大的坑
以上所有讲的都是在本地运行自动化构建，配置低的小伙伴不高兴了，打包📦炒鸡卡，于是将项目部署到服务器上了
然而问题来了，我在服务器上运行时发现构建无法成功，提示`pod: command not found`，于是google了一堆资料，多种方式都没解决，才想到是Linux服务器并没有配置相关的环境，当然不支持`brew`、`rvm`、`gem`、`pod`、`bundle`、`fastlane`这些命令啦，查询stackoverflow告诉我![](http://upload-images.jianshu.io/upload_images/4483590-43bd347d571a9e92.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
蛤？相关的命令行工具只能在Mac环境运行？
![](http://upload-images.jianshu.io/upload_images/4483590-aed0537e4ed4c2cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以你需要一台Mac服务器！
故而我现在本地只能手动点构建或者是采用定时构建方式，在开机后自动启用Jenkins
而服务器端虽然可以hook到git commit log，但是无法通过以上脚本运行，干脆就作为邮件发送器算了😄

# 参考资料
[手把手教你利用Jenkins持续集成iOS项目](https://www.jianshu.com/p/41ecb06ae95f)
[利用 Jenkins 实现 iOS 自动化构建踩坑记录](http://tofu.wiki/2016/12/04/Jenkins-Series-of-tutorials-Part-1/)
[Mac中jenkins的使用——自动构建](http://blog.csdn.net/potato512/article/details/52289136)
[最全Jenkins+SVN+iOS+cocoapods环境搭建及其错误汇总](http://www.bijishequ.com/detail/542335?p=)
[Jenkins的几个问题](http://shengshui.com/?p=1848)
[jenkins 邮件配置展示change信息](http://blog.csdn.net/www203203/article/details/55509499)
[Jenkins进阶系列之——02email-ext邮件通知模板](http://blog.csdn.net/limm33/article/details/51191080)
