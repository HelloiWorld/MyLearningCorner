# 读《Effective Objective-C 2.0：编写高质量iOS与OS X代码的52个有效方法》有感
***

### 1. Objective-C使用动态绑定的消息结构，在运行时才会检查对象类型。
这种动态消息工作方式决定了其不可能实现真正的私有方法或私有实例变量。

### 2. 在类的头文件中尽量少引入其他头文件
* 使用`@class xxx;`（向前声明）的方式在头文件导入新类，并在实现文件中再`#import`该类的头文件。这样将引入头文件的时机尽量延后，只在**确有需要**时才引入，既尽可能的降低了类之间的耦合，又减少了类的使用者所需引入的头文件数量从而减少编译时间。
* 向前声明解决了两个类互相引用的问题。

### 3. 多用字面量语法，少用与之等价的方法
* 使用字面量语法可以缩减源代码长度，使其更为易读。
* 字面量语法创建数组、字典对象更为安全，因为若存在空值对象则会抛出异常，有助于查错。
* 字面量语法创建出的字符串、数组、字典对象都是不可变的，要创建可变对象还需额外调用`[obj mutableCopy]`方法。

### 4. 多用类型常量，少用#define预处理命令
* `static` + `const`声明的变量作用相当于`#define`，但会带有类型信息
* `const`右边的总不能被修改，所以正确写法是：

	    static NSString * const coder = @"Hello world!";
	    static const NSTimeInterval duration = .3f;
* `extern` + `const`声明并定义全局常量的写法是：

		//.h文件中公开，也可只定义在引入其头文件的类中
   	    extern NSString *const NotificationUserLogout;
   	    //.m
	    NSString *const NotificationUserLogout = @"NotificationUserLogout";
	
### 5. 用枚举表示状态、选项、状态码
* 使用`NS_OPTIONS`宏定义枚举类型，可以通过按位或操作将多个选项同时组合起来
* 处理枚举类型的`switch`语句中不要实现`default`分支，应覆盖所有可能出现的情况

### 6. 理解“属性特质”
* 属性特质
 * 原子性
     * `atomic`
     * `nonatomic`
 * 读写权限
     * `readwrite`
     * `readonly`
 * 内存管理语义
     * `assign`
     * `strong`
     * `weak`
     * `unsafe_unretained`
     * `copy`
 * 方法名
     * `getter=<name>`
     * `setter=<name>`
* 对应关键字
 * `@synthesize`：未手动实现`setter`和`getter`方法时编译器会自动生成，也可用于修改实例变量名（去`_`，不推荐） 
 * `@dynamic`：不自动生成`getter`和`setter`方法

### 7. 在对象内部尽量访问实例变量
> * 由于不经过Objective-C的方法派发，直接访问实例变量的的速度更快
* 直接访问实例变量不会调用其设置方法，故绕过了内存管理语义，也不会触发KVO通知

* 在对象内部读取数据时，应该直接通过实例变量来读，而写入数据时，则应通过属性来写
* 在初始化方法及`dealloc`方法中，总是应该直接通过实例变量(`_xx`)来读写数据
* 使用懒加载时，要通过属性（`self.xx`）来读取数据

### 8. 理解“对象等同性”
> * `==`比较的是指针本身
* `isEqual:`默认比较的是指针地址
  1. 先判`self == object`，指针相等则返回`YES`
  2. 再判`[self class] != [object class]`，若所属类不相同则直接返回`NO`（注：需要考虑继承情况，可使用`isKindOfClass:`）
  3. 依次分别对每个属性做值判断，一旦有一个不等则返回`NO`
  4. 最后默认返回`YES`
* 等同性约定：若`isEqual:`判定两对象相等，则`hash`一定相等；但`hash`相同的两对象未必相等

* 若要检测对象的等同性，请提供`isEqual:`与`hash`方法
* 重写`hash`方法时，应提供计算速度快而且哈希码碰撞几率低的算法，不要创建额外的开销

### 9. 以“类簇模式”隐藏实现细节
> 类簇模式可以把实现细节隐藏在公共接口之后，在系统框架中经常使用。

Objective-C语言无法指明某个基类是“抽象的”，所以子类是不知道必须要实现父类的某个方法的。开发中常常是在超类的开发文档中注明。
个人的做法是在父类抽象方法实现中加

	#define MustOverride() @throw [NSException exceptionWithName:NSInvalidArgumentException reason:[NSString stringWithFormat:@"%s must be overridden in a subclass/category", __PRETTY_FUNCTION__] userInfo:nil]
	
宏，这样一旦子类实例将消息发送到了父类该方法就会抛出异常

### 10. 在既有类中使用关联对象存放自定义数据
	void objc_setAssociatedObject(id object, void *key, id value, objc_AssociationPolicy)
	id objc_getAssociatedObject(id object, void *key)
	void objc_removeAssociatedObjects(id object)
关联对象涉及到runtime的一些用法，需要引入`#import <objc/runtime.h>`，常常用于给分类(category)增加属性

### 11. 理解消息发送(objc_msgSend)的作用
关于消息发送的相关知识，具体可查阅[《招聘一个靠谱的iOS》面试题参考答案（上）：objc中向一个对象发送消息obj-foo和objc_msgsend函数之间有什么关系](https://github.com/ChenYilong/iOSInterviewQuestions/blob/master/01《招聘一个靠谱的iOS》面试题参考答案/《招聘一个靠谱的iOS》面试题参考答案（上）.md#17-objc中向一个对象发送消息obj-foo和objc_msgsend函数之间有什么关系)

### 12. 理解消息转发(objc_msgForward)机制
关于消息发送的相关知识，具体可查阅[《招聘一个靠谱的iOS》面试题参考答案（下）：objc_msgforward函数是做什么的直接调用它将会发生什么](https://github.com/ChenYilong/iOSInterviewQuestions/blob/master/01《招聘一个靠谱的iOS》面试题参考答案/《招聘一个靠谱的iOS》面试题参考答案（下）.md#25-_objc_msgforward函数是做什么的直接调用它将会发生什么)

### 13. “方法调配技术”(method swizzling)
实质就是交换两个方法的实现，可用来在运行期为已有类新增或替换选择子(`@selector`)所对应的方法。比如在分类的`load`方法中，执行方法交换，以修改替换原有的方法实现。

### 14. 理解“类对象”结构
	typedef struct objc_class *Class;
	struct objc_class {
		Class isa; //isa指针，指向metaclass（该类的元类）
		Class super_class; //指向objc_class（该类）的super_class（父类）
		const char *name; //objc_class（该类）的类名
		long version; //objc_class（该类）的版本信息，初始化为0，可以通过runtime函数class_setVersion和class_getVersion进行修改和读取
		long info; //一些标识信息，如CLS_CLASS表示objc_class（该类）为普通类。ClS_CLASS表示objc_class（该类）为metaclass（元类）
		long instance_size; //objc_class（该类）的实例变量的大小
		struct objc_ivar_list *ivars; //用于存储每个成员变量的地址
		struct objc_method_list **methodLists; //方法列表，与info标识关联
		struct objc_cache *cache; //指向最近使用的方法的指针，用于提升效率
		struct objc_protocol_list *protocols; //存储objc_class（该类）的一些协议
	};
	
### 15. 用前缀避免命名空间冲突
Objective-C没有内置命名空间(namespace)机制。

### 16. 提供“全能”(指定，designated)初始化方法
指定初始化方法应覆盖所有其他初始化方法，并覆写超类对应方法。

### 17. 实现description方法
	#import <objc/runtime.h> //导入runtime头文件
	
	//重写description方法同下，NSLog打印model详细信息
	//重写debugDescription, lldb打印model详细信息
	- (NSString *)debugDescription {
	    //声明一个字典
	    NSMutableDictionary *dictionary = [NSMutableDictionary dictionary];
	    
	    //得到当前class的所有属性
	    uint count;
	    objc_property_t *properties = class_copyPropertyList([self class], &count);
	    
	    //循环并用KVC得到每个属性的值
	    for (int i = 0; i < count; i++) {
	        objc_property_t property = properties[i];
	        NSString *name = @(property_getName(property));
	        id value = [self valueForKey:name]?:@"nil";//默认值为nil字符串
	        [dictionary setObject:value forKey:name];//装载到字典里
	    }
	    
	    //释放
	    free(properties);
	    
	    return [NSString stringWithFormat:@"<%@: %p> -- %@",[self class],self,dictionary];
	}
	
### 18. 尽量使用不可变对象
* 不希望外部修改的属性用`readonly`语义声明，若内部可修改设置对应的`readwrite`属性
* 不宜公开可变的collection属性，封装方法返回collection也最好先转成对应的不可变属性

### 19. 使用清晰而协调的命名方式
代码洁癖的自我修养

### 20. 为私有方法名加前缀
有助于区分于公开方法，一目了然

### 21. 理解Objectiove-C错误模型
* 非严重错误无须使用异常，必须使用时参考[第9条]()添加异常宏
* 使用`NSError`对象封装错误信息，并返回给调用者

### 22. 理解NSCopying协议
这是一个**严谨**的单例模式宏，支持ARC和MRC

	#if __has_feature(objc_arc) // ARC

	#define DEF_SINGLETON(name) \
	static id _instance; \
	+ (id)allocWithZone:(struct _NSZone *)zone \
	{ \
	static dispatch_once_t onceToken; \
	dispatch_once(&onceToken, ^{ \
	_instance = [super allocWithZone:zone]; \
	}); \
	return _instance; \
	} \
	\
	+ (instancetype)sharedInstance \
	{ \
	static dispatch_once_t onceToken; \
	dispatch_once(&onceToken, ^{ \
	_instance = [[self alloc] init]; \
	});\
	return _instance; \
	} \
	+ (id)copyWithZone:(struct _NSZone *)zone \
	{ \
	return _instance; \
	}
	
	#else // 非ARC
	
	#define DEF_SINGLETON(name) \
	static id _instance; \
	+ (id)allocWithZone:(struct _NSZone *)zone \
	{ \
	static dispatch_once_t onceToken; \
	dispatch_once(&onceToken, ^{ \
	_instance = [super allocWithZone:zone]; \
	}); \
	return _instance; \
	} \
	\
	+ (instancetype)sharedInstance \
	{ \
	static dispatch_once_t onceToken; \
	dispatch_once(&onceToken, ^{ \
	_instance = [[self alloc] init]; \
	}); \
	return _instance; \
	} \
	\
	- (oneway void)release \
	{ \
	\
	} \
	\
	- (id)autorelease \
	{ \
	return _instance; \
	} \
	\
	- (id)retain \
	{ \
	return _instance; \
	} \
	\
	- (NSUInteger)retainCount \
	{ \
	return 1; \
	} \
	\
	+ (id)copyWithZone:(struct _NSZone *)zone \
	{ \
	return _instance; \
	}
	
	#endif

### 23. 通过委托与数据源协议进行对象间通信
* 对于经常要使用到的代理(如`UITextFieldDelegate`,`UITextViewDelegate`,`UITableViewDataSource`等)，可交由一个封装的对象专门去实现这些方法，以节省重复代码的书写。
* **进一步优化**：如需频繁判断`[_delegate respondsToSelector:@selector(xx)]`的结果，可考虑实现含有位段的结构体，将这一判断结果缓存至其中

### 24. 将类的实现代码分散到便于管理的数个分类之中
现在已采用的方法，为大文件"减负"，如`AppDelegate+Configuration`

### 25. 总是为第三方类的分类名称加前缀
分类名和方法名都宜添加前缀，防冲突

### 26. 勿在分类中声明属性
有必要的时候还是用关联对象实现吧，虽然作者不推荐

### 27. 使用"class-continuation分类"隐藏实现细节
其实也就是在.m文件中声明私有属性，但并不是真的私有，仍然可以用runtime遍历属性列表找到该属性。

### 28. 通过协议提供匿名对象
通过`id<xxxDelegate>`这种方式声明匿名对象，这样使用者无须关心这种对象的具体类型，只专注于它能去做的事即可。
#### 架构设计
比如要同时支持百度地图，高德地图，谷歌地图，这时由于这些第三方库来自不同的类，所以无法继承自同一基类，但它们都具有类似的特征，故可以采用这种协议方法将类似功能的方法提取公开，结合类簇模式思想调取对应第三方库的实现。

### 29. 理解引用计数
> * retain: **递增**保留计数
* release: **递减**保留计数
* autorelease: 待**稍后**(通常是下一次"事件循环",event loop)清理自动释放池(autorelease pool)时，再递减保留计数

**保留计数至少为1。**若保留计数为正，则对象继续存活；若保留计数降为0，对象就被销毁。

### 30. 以ARC简化引用计数
> * ARC->MRC：`-fno-objc-arc`
* MRC->ARC：`-fobjc-arc`

以`alloc`，`new`，`copy`，`mutableCopy`作为前缀的方法（这里要注意的是，后面接的词语首位必须是大写字母，即符合驼峰命名规则，详见《Objective-C高级编程：iOS与OS X多线程和内存管理》此书），其返回的对象"归调用者所有"(调用这四种方法的那段代码要负责释放方法所返回的对象，即autorelease不生效，需要手动额外抵消一次保留操作)。

### 31. 在dealloc方法中只释放引用并解除监听
ARC下`dealloc`方法中应该做的

* 打印销毁的日志（判断此方法是否执行）：`NSLog(@"dealloc: %@",[self class]);`
* 移除KVO：`[_scrollView removeObserver:self forKeyPath:@"contentOffset"];`
* 移除通知： `[[NSNotificationCenter defaultCenter] removeObserver:self];`
* 释放CoreFoundation对象：`CFRelease(coreFoundationObject)`
* 不推荐清理文件资源，常见的如数据库、播放器等等，关闭它们更应该明确其生命期进行管理，而不是延迟到**并不一定会执行**的`dealloc`方法
* 不推荐在此关闭定时器（正确的定时器移除应该由生命周期方法控制，打破循环引用）：`[_timer invalidate]`（保险做法见[第52条]()）

### 32. 编写“异常安全代码”时留意内存管理问题
* ARC下使用`@try...@catch...@finally`实际上会造成`try`块内所创立的对象内存泄漏，所以往往不这么干
* ARC下打开异常需打开编译器的`-fobjc-arc-exceptions`标志，副作用是使应用程序变大且降低运行效率
* 异常的处理方法是使用`NSError`传回错误信息，调用者根据情况作相应处理

### 33. 以弱引用避免保留环
    __weak typeof(self) weakSelf = self;
    self.block = ^{
        __strong typeof(self) strongSelf = weakSelf;
        if (strongSelf) {
        	// use strongSelf
        }
    };


Weak-Strong-Dance防止block和对象间的循环引用不用多说，偷懒的写法是使用宏：

	#ifndef weakify
	#if __has_feature(objc_arc)
	
	#define weakify( x ) \\
	_Pragma("clang diagnostic push") \\
	_Pragma("clang diagnostic ignored \\"-Wshadow\\"") \\
	autoreleasepool{} __weak __typeof__(x) __weak_##x##__ = x; \\
	_Pragma("clang diagnostic pop")
	
	#else
	
	#define weakify( x ) \\
	_Pragma("clang diagnostic push") \\
	_Pragma("clang diagnostic ignored \\"-Wshadow\\"") \\
	autoreleasepool{} __block __typeof__(x) __block_##x##__ = x; \\
	_Pragma("clang diagnostic pop")
	
	#endif
	#endif
	
	#ifndef strongify
	#if __has_feature(objc_arc)
	
	#define strongify( x ) \\
	_Pragma("clang diagnostic push") \\
	_Pragma("clang diagnostic ignored \\"-Wshadow\\"") \\
	try{} @finally{} __typeof__(x) x = __weak_##x##__; \\
	_Pragma("clang diagnostic pop")
	
	#else
	
	#define strongify( x ) \\
	_Pragma("clang diagnostic push") \\
	_Pragma("clang diagnostic ignored \\"-Wshadow\\"") \\
	try{} @finally{} __typeof__(x) x = __block_##x##__; \\
	_Pragma("clang diagnostic pop")
	
	#endif
	#endif
	
RAC里的无敌写法：

	#define weakify(...) \\
	    autoreleasepool {} \\
	    metamacro_foreach_cxt(rac_weakify_,, __weak, __VA_ARGS__)
	
	#define strongify(...) \\
	    try {} @finally {} \\
	    _Pragma("clang diagnostic push") \\
	    _Pragma("clang diagnostic ignored \\"-Wshadow\\"") \\
	    metamacro_foreach(rac_strongify_,, __VA_ARGS__) \\
	    _Pragma("clang diagnostic pop")
	    
### 34. 以“自动释放池块”降低内存峰值
常用于for循环中，将循环体代码用`@autoreleasepool{...}`包裹起来，可以及时在每次循环后降低内存峰值，而不用等到线程执行下一次事件循环时

### 35. 用“僵尸对象”调试内存管理问题
> * 开启方式：Xcode -> Edit Scheme -> Run -> Diagnostics -> Enable Zombie Objects<br/
* 原理：修改对象的isa指针，令其指向特殊的僵尸类，此僵尸类能够响应所有的选择子（通过“完整的消息转发机制”）。<br/>
* 响应方式：如果消息接收者是僵尸对象（名称前缀为\_NSZombie\_），此时打印一条包含消息内容及其接收者的消息，然后终止程序。

调试时通过环境变量NSZombieEnabled开启此功能，系统将不再回收对象，而是将其转化为僵尸对象。

### 36. 不要使用retainCount
`retainCount`**可能永远不返回0。**因为有时系统会优化对象的释放行为，当保留计数为1且执行了递减操作，这时为了节省对象引用计数-1的开销，会在稍后的某个时间点直接回收。
> * 系统会尽可能把`NSString`实现成单例对象。如果是编译期常量，编译器会把`NSString`对象所表示的数据放到应用程序的二进制文件里，这样运行时无需再创建`NSString`对象。
* `NSNumber`类似使用了“标签指针”（把与数值有关的全部消息都放在指针值里面）来标注特定类型（不包括浮点数对象）的数值，同样在运行时就无需再创建`NSNumber`对象。
* 进行了以上两种方式优化的单例对象，其保留计数绝对不会变（很大的值）。这种对象的保留及释放都是空操作。

### 37. 理解“块”
> 语法结构：`return_type (^block_name)(parameters)`

* 块是一种词法闭包，与python中的匿名函数`lambda`不一样，它可以无返回值，也可以作为传参使用
* 默认情况下块所捕获的变量是不可以在块里修改的
* 使用`__block`关键字修饰的变量可以在块内部被修改
* **块总能修改实例变量**，所以声明时无须加`__block`
* 通过读取或写入操作使块捕获了实例变量，此时`self`变量也自动被捕获了（同时也会将其保留），如果此时`self`所指代的那个对象同时保留了块，这种情况就导致了“保留环”(retain cycle)。

### 38. 为常用的块类型创建typedef
使用简单，作为属性时用`copy`修饰

### 39. 用handler块降低代码分散程度
建议用同一个handler块来处理网络成功与失败情况，比如虽然网络请求成功了，但数据不符合预期，这时方便将错误信息用`NSError`一并返回

### 40. 用块引用其所属对象时不要出现保留环
* 正常使用单例模式的网络获取器一般不会出现保留环，它运行在调用者所在的队列上，所以不用过于担心的在返回的块内加`@weakify...@strongify...`或`dispatch_async(dispatch_get_main_queue(), ^{...})`这样的代码；
* 但如果在调用者内部使用`self.completionHandler = completion`这样的代码将块持有了(之所有这样做的原因是为了稍后使用)，这时往往要考虑保留环的问题，可在`_completionHandler()`执行后使用`self.completionHandler = nil`方式将其清除。

### 41. 多用派发队列，少用同步锁
* 关于锁的相关知识可以查看[iOS中的锁——由属性atomic想到的线程安全](https://www.jianshu.com/p/0b40a63c6436)
* GCD是可以高效的代替同步块或锁对象。属性通常不设置`atomic`的原因在于它并非真正的线程安全，读取操作可以并行，但写入操作应该限定单独执行。
		
		//指定全局并发队列，若是跑在主线程上会出现死锁
	    _syncQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT,0);
		
		- (NSUInteger)ticketCount {
			__block NSUInteger localTicketCount;
			//不开新线程，顺序执行
			dispatch_sync(_syncQueue, ^{
				localTicketCount = _ticketCount;
			});
			return localTicketCount;
		}
		
		- (void)setTicketCount:(NSUInteger)ticketCount {
			//等待所有并发块都执行完毕，才会单独执行块内代码
			dispatch_barrier_async(_syncQueue, ^{
				_ticketCount = ticketCount;
			});
		}
		
### 42. 多用GCD，少用performSelector方法
* `performSelector`系列方法使得编译器无法在编译器就确定选择子和返回值，故ARC下并没有插入内存管理方法，易造成内存泄漏。
* `performSelector`系列方法参数的局限性太大，都有可替代的方案。

### 43. 掌握GCD及操作队列的使用时机
`NSOperation`是对GCD更高层次的封装，也提供了GCD无法实现（很难实现）的特性。关于它及`NSOperationQueue`的用法可参照 [AFNetworking](https://github.com/AFNetworking/AFNetworking)

### 44. 通过Dispatch Group机制，根据系统资源状况来执行任务
通过dispatch group可以在并发队列同时执行多项任务，并在这组任务执行完毕时获得通知。

### 45. 使用dispatch_once来执行只需运行一次的线程安全代码
	+ (instancetype)sharedInstance { 
		static ManageClass *sharedInstance = nil; //每次执行会复用变量
		static dispatch_once_t onceToken; 
		dispatch_once(&onceToken, ^{ 
			sharedInstance = [[self alloc] init]; 
		});
		return sharedInstance; 
	} 

编写“只需执行一次的线程安全代码”，`dispatch_once`方式实现单例性能显然要高于同步锁机制`@synchronized`

### 46. 不要使用dispatch\_get\_current\_queue
* 理解死锁相关的模型
* `dispatch_get_current_queue`仅应该作为调试使用
* 队列间有层级关系，排在某条队列中的块会在其上级队列里执行。

### 47. 熟悉系统框架
`Foundation`、`CoreFoundation`提供的API多了解一下

### 48. 多用块枚举，少用for循环
* for循环：优点是方便取下标，缺点是会多创建一个中介数组，额外增加对附加对象的创建和释放操作
* 快速枚举：高效，不允许对遍历集合修改，获取当前下标还需额外定义变量
* 块枚举：提供更完善的块信息，也可修改块签名，无须另行编码就能并发执行遍历操作

        - (void)enumerateObjectsUsingBlock:(void(^)(id object, NSUInteger idx, BOOL *stop))block
        - (void)enumerateKeysAndObjectsUsingBlock:(void(^)(id key, id object, BOOL *stop))block

### 49. 对自定义其内存管理语义的collection使用无缝桥接
> * `__bridge`：OC和CF对象转化时只涉及对象类型不涉及对象所有权转化，由发起转换对象控制。
* `__bridge_transfer`：CF对象转化成OC对象时，移交其所有权，由ARC管理，作用同`CFBridgingRelease()`
* `__bridge_retained`：OC对象转化成CF对象，移交其所有权，由CF对象调用相应的`CFRelease()`方法销毁，作用同`CFBridgingRetain()`

### 50. 构建缓存时使用NSCache而非NSDictionary
> `NSCache`优点在于当系统资源将要耗尽时，它会自动删减“最久未使用的”(LRU)缓存。

所以我选择用 [YYCache](https://github.com/ibireme/YYCache)

### 51. 精简initialize与load的实现代码
* `+ (void)load`：运行期（通常是应用程序启动）每个类及分类必定且只有一次会调用此方法。无法覆写，常用于method swizzling，尽量不在里面做多余操作
* `+ (void)initialize`：懒加载模式，首次使用该类之前调用且只调用一次。其未覆写此方法的子类也会收到超类消息

### 52. 别忘了NSTimer会保留其目标对象
`NSTimer`会保留目标，反复执行的计时器会造成保留环是个老生常谈的问题了，常常用的方法时创建一个分类，使用“块”来打破保留环

	+ (NSTimer *)weak_scheduledTimerWithTimeInterval:(NSTimeInterval)timeInterval
                                         repeats:(BOOL)repeats
                                    handlerBlock:(void(^)(void))handler {
	    return [self scheduledTimerWithTimeInterval:timeInterval
	                                         target:self
	                                       selector:@selector(handlerBlockInvoke:)
	                                       userInfo:[handler copy]
	                                        repeats:repeats];
	}
	
	+ (NSTimer *)weak_timerWithTimeInterval:(NSTimeInterval)timeInterval
	                                repeats:(BOOL)repeats
	                           handlerBlock:(void (^)(NSTimer *timer))handler {
	    return [NSTimer timerWithTimeInterval:timeInterval
	                                   target:self
	                                 selector:@selector(handlerBlockInvoke:)
	                                 userInfo:[handler copy]
	                                  repeats:repeats];
	}
	
	+ (void)handlerBlockInvoke:(NSTimer *)timer {
	    void (^block)(void) = timer.userInfo;
	    if (block) {
	        block();
	    }
	}
