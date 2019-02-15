# 面试常见问题整理
1. [关于APP内购，怎样处理掉单问题？](#关于APP内购，怎样处理掉单问题？)
2. [手指点击屏幕，中间发生了哪些过程？响应链传递机制？](手指点击屏幕，中间发生了哪些过程？响应链传递机制？)
3. [关于lldb调试命令你知道多少？](#关于lldb调试命令你知道多少？)
4. [runloop与线程的关系？](#runloop与线程的关系？)
5. [runtime了解多少？](#runtime了解多少？)
6. [block为什么不能修改外部变量？__block起到的作用是什么？](#block为什么不能修改外部变量？__block起到的作用是什么？)
7. [谈谈对于浅拷贝和深拷贝的理解](#谈谈对于浅拷贝和深拷贝的理解)
8. [UIView和CALayer之间的关系？](#UIView和CALayer之间的关系？)
9. [细述苹果的消息推送原理和过程](#细述苹果的消息推送原理和过程)
10. [开发中常见的加密方式原理及应用](#开发中常见的加密方式原理及应用)


## 1. 关于APP内购，怎样处理掉单问题？
支付过程如下：  
1. 调用IAP接口发起支付  
2. 支付成功，获取App Receipt票据，调用充值接口验证  
3. 验证通过，给用户充值虚拟商品并回调给App  

掉单可能会出现的场景：  
1. 完成第1步操作后，App进程因为某种原因被kill了，但可能已经扣款成功  
2. 完成第2步操作拿到票据后，由于网络原因与服务器通信失败，尚未验证票据  
3. 服务器与苹果服务器之间验证失败，需要后台设置重验逻辑  

> `[SKPaymentQueue defaultQueue]`这个队列里面存着所有的已支付、未支付的订单，而且需要手动移除，而APP每次启动的时候都会去判断这个队列里面是否为空，如果不为空的话会调用`<SKPaymentTransactionObserver>`代理的`-(void)paymentQueue:(SKPaymentQueue *)queue updatedTransactions:(NSArray *)transactions`方法，在验证成功之后移除队列`[[SKPaymentQueue defaultQueue] finishTransaction: transaction];`

应对处理如下：  
将支付凭证及相关信息存到本地，然后再去请求服务器验证，这时会出现两种情况：  
1）如果用户未卸载APP，那么在每次启动APP的时候都应该去检查是否有保存的票据信息，如果有，依次轮询交由服务器验证，验证成功之后将相应票据信息和对应事务移除  
2）如果中途用户由于卸载了APP致使本地存储丢失，沙盒里的票据信息也没了，所以要对上诉方案完善，在购买前由服务器先保存一份商品订单，用户凭借支付凭证与这份下单表作对比，如果确实为准备支付而且凭证属实，则为用户恢复数据



## 2. 手指点击屏幕，中间发生了哪些过程？响应链传递机制？
1. **生成事件**。当用户点击屏幕时，会产生一个触摸事件，并放入由Application管理的事件队列中，然后在队列中取出最前面的事件交给Window处理。
2. **查找第一响应者对象**。Window收到事件后会在视图层次结构中找到最适合的一个视图来处理事件，通常一个窗口中最适合处理当前事件的对象称为第一响应对象。
3. **处理事件**。通常最后是第一响应对象处理事件，如果第一响应对象无法处理事件，就会把事件传递给下一个响应对象，直到Application。如果Application也无法处理，那就丢弃掉此事件。  
当Application知道了第一响应对象后，就会把事件交给第一响应对象来处理，如果第一响应对象能顺利处理事件，则整个响应结束，但是第一响应对象如果无法处理事件，就会把事件传递给下一个响应对象（nextResponder），一直沿着响应链向上回溯。  
**view -> ViewController -> window -> Application -> 丢弃**


## 3. 关于lldb调试命令你知道多少？
> `expression`(可简写为`exp`/`expr`)命令是执行一个表达式，并将表达式返回的结果输出，是LLDB调试命令中最重要的命令，也是我们常用的`p`和`po`命令的鼻祖

如果提示`Unable to call function “objc_msgSend” at 0x1e7e08c: no return type information available.`这种错误，其实是要使用强制转义增加返回值类型。

#### `p` & `print` & `e` & `call` 命令
这几个命令其实就是`“expression -- ”`的别名

#### `po`命令 
oc里所有的对象都是用指针表示的，打印出来的是对象的指针，而不是对象本身，可以采用`-o`来打印对象本身。为了更加方便的使用，LLDB为`“expression -o —“`定了一个别名：`po`


## 4. runloop与线程的关系
1. 主线程的run loop是自动创建并默认启动的。main.m中`UIApplicationMain()`函数会为main thread设置一个NSRunLoop对象，以保证App在休眠时接收到用户触摸事件也能响应。
2. 对于子线程或其他线程，run loop默认不启动，在第一次调用时才会去创建，并在线程结束时销毁。
3. 可以通过`[NSRunLoop currentRunLoop]`获取当前线程的run loop

## 5. runtime了解多少？
消息传递和消息转发先省略不写。runtime常见应用如下：

* 关联对象(Objective-C Associated Objects)给分类增加属性
* 方法魔法(Method Swizzling)方法添加和替换和KVO实现
* 消息转发(热更新)解决Bug(JSPatch)
* 实现NSCoding的自动归档和自动解档
* 实现字典和模型的自动转换(MJExtension)

## 6. block为什么不能修改外部变量？__block起到的作用是什么？
**block中不允许修改外部变量的值**。这里所说的外部变量的值，指的是栈中指针的内存地址。`__block`所起到的作用就是只要观察到该变量被 block 所持有，就将“外部变量”在栈中的内存地址放到了堆中。进而在block内部也可以修改外部变量的值。

## 7. 谈谈对于浅拷贝和深拷贝的理解
#### OC中的深拷贝，浅拷贝
* 对于不可变对象，copy都是浅复制，即指针复制。mutableCopy都是内存复制，即深复制 
* 对于可变对象，copy和mutablecopy一般是内存复制，即深复制 
* 容器类对象，不论是可变的还是不可变的，copy，mutableCopy返回的对象里所包含的对象的地址和之前都是一样的，即容器内对象都是浅拷贝。

#### swift中深拷贝，浅拷贝
* class是引用类型的，对于class的拷贝是浅拷贝
* struct是值类型的，对于struct的拷贝是深拷贝

## 8. UIView和CALayer之间的关系
1. UIView可以响应事件，CALayer不可以。其根源是UIView是继承自UIResponder（UIResponder中定义了处理各种事件和事件传递的接口）的，而CALayer是继承自NSObject的，并没有响应的处理事件的接口。
2. UIView是CALayer的delegate，其layer属性返回了所在CALayer的实例。YYAsyncLayer就是通过重写`override class var layerClass: AnyClass {
        return YYAsyncLayer.self
    }`修改其绘制内容的。
3. UIView主要处理事件，CALayer负责绘制内容。界面性能优化中有利用此机制，对于不需要响应用户事件的地方直接使用CALayer更加节省性能。
4. 访问UIView坐标相关的属性其实是访问它所在CALayer的属性，特别的是CALayer中多出`anchorPoint`属性，使用CGPoint结构表示，值域是0~1，是个比例值。更改它即会修改layer的position的位置。


## 9. 细述苹果的消息推送原理和过程
1. 为应用程序申请消息推送服务。在App启动时设备向APNs服务器发送注册请求
2. APNs服务器接受请求，并将deviceToken返给你设备上的应用程序
3. 客户端应用程序将deviceToken发送给后台服务器程序，后台接收并储存
4. 后台服务器向APNs服务器发送推送消息
5. APNs服务器将消息发给deviceToken对应设备上的应用程序
![](https://upload-images.jianshu.io/upload_images/1340708-47499ef73a24d52f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)


## 10. 开发中常见的加密方式原理及应用
#### Base64编码方式
##### 原理： 
1. 将所有字符转化为ASCII码;   
2. 将ASCII码转化为8位二进制;  
3. 将二进制3个归成一组(不足3个在后边补0)共24位，再拆分成4组，每组6位;  
4. 统一在6位二进制前补两个0凑足8位;（每两个0用一个=表示）  
5. 将补0后的二进制转为十进制;  
6. 从Base64编码表获取十进制对应的Base64编码;  

#### MD5加密
> 哈希（散列）函数：单向加密算法，常见有MD5、SHA

##### 特点：
* 不可逆运算，但可能被暴力破解
* 对不同的数据加密的结果是定长的32位字符（不管文件多大都一样）
* 对相同的数据加密，得到的结果是一样的
* 抗修改性: 对原数据进行任何改动，哪怕只修改一个字节，所得到的MD5值都有很大区别.
* 弱抗碰撞: 已知原数据和其MD5值，想找到一个具有相同MD5值的数据(即伪造数据)是非常困难的.
* 强抗碰撞: 想找到两个不同数据，使他们具有相同的MD5值，是非常困难的

鉴于Hash算法加密结果固定，可能因为碰撞破解，往往采用**加盐**(salt，在原始数据拼接**固定不变的**足够复杂的随机字符串)的方式提升安全性。

#### AES对称加密
> 对称加密：加密和解密使用同一个非公开的密钥。常见有DES、3DES、AES三种

##### 特点：
* 高级加密标准，加密速度快效率高
* 只要是对称加密都有**ECB**（电子密码本，每个块独立加密）和**CBC**（密码块链，使用一个密钥和一个初始化向量(IV)对数据执行加密转换）加密模式，常用CBC模式

#### RSA非对称加密
> 非对称加密：需要成对的两个密钥，公开密钥(public key)和私有密钥(private key)。

##### 特点：
* 加解密速度慢
* 通常数据本身的加密和解密使用对称加密算法(AES)，用RSA算法加密并传输对称算法所需的密钥。

#### 数字签名/数字证书
详请阅读：

* [阮一峰：数字签名是什么？](http://www.ruanyifeng.com/blog/2011/08/what_is_a_digital_signature.html)
* [浅析数字签名和数字证书](https://www.jianshu.com/p/9cba08499258)
