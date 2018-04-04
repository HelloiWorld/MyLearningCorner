# 读《Objective-C高级编程：iOS与OS X多线程和内存管理》有感

# 1. 自动引用计数
## 内存管理/引用计数
#### 内存管理的思考方式
* 自己生成的对象，自己所持有（以`alloc/new/copy/mutableCopy`名称开头，符合驼峰命名规则的方法名，也遵循此规则）
* 非自己生成的对象，自己也能持有
* 不再需要自己持有的对象时释放
* 非自己持有的对象无法释放

对象操作  | Objective-C 方法
------------- | -------------
生成并持有对象  | `alloc/new/copy/mutableCopy`方法
持有对象  | `retain`方法
释放对象  | `release`方法
废弃对象  | `dealloc`方法

#### 引用计数计算规则
* 调用`alloc`或`retain`方法后，引用计数值加1
* 调用`release`后，引用计数值减1
* 引用计数为0时，调用`dealloc`方法废弃对象

#### IMP Caching
> 在进行方法调用时，为了解决类名/方法名以及取得方法运行时的函数指针，要在框架初始化时对其结果值进行缓存，实际的方法调用就是使用缓存的结果值。

比如类对象结构体中的`objc_cache *cache`就是指向最近使用的方法的指针，以提升方法调用效率。

## ARC规则
#### 所有权修饰符
修饰符 | 特性 | 作用
------------- | ------------- | -------------
`__strong` | id类型和对象类型**默认**的所有权修饰符 | 被修饰的变量在超出其变量作用域时，即在该变量被废弃时，会释放其被赋予的对象
`__weak` | 1.不能持有对象实例<br/> 2.在其持有对象被废弃时将自动失效且赋值为nil | 解决循环引用造成的“内存泄漏”（应当废弃的对象在超出其生存周期后继续存在）问题
`__unsafe_unretained` | 1.编译器不对其变量进行内存管理<br/> 2.自己生成并持有的对象不能为自己所有<br/> 3.不同于`__weak`之处在于对象被废弃时不会自动清空，造成“悬垂指针” | 1.避免`__weak`遍历weak表带来的性能开销<br/> 2.替代`__weak`在其无法生效的版本(iOS4及OS X Snow Leopard)
`__autoreleasing` | 1.id的指针或对象的指针没有显式指定时会被附加上此修饰符，如Cocoa框架中`NSError`对象作为方法参数返回值的情况<br/> 2.显式指定时必须是自动变量（包括局部变量、函数以及方法参数） | 等价于在ARC无效时调用对象的`autorelease`方法，即对象被注册到`autoreleasepool`
> 注：调试注册到`autoreleasePool`上的对象的非公开函数：`_objc_autoreleasePoolPrint()`

#### 规则
* 不能使用`retain/release/retainCount/autorelease`：设置ARC有效时，无须再次键入`retain`或`release`代码
* 不能使用`NSAllocateObject/NSDeallocateObject`：`alloc`的实现实际上是直接调用`NSAllocateObject`函数来生成并持有对象的，故应禁止底层调用，后者同理
* 须遵守内存管理的方法命名规则：除了`alloc/new/copy/mutableCopy`开头的驼峰式方法命名规则外，ARC有效时还追加了`init`驼峰命名规则，并且此`init`开头的方法必须要有返回对象
* 不要显式调用`dealloc`：对象被废弃时，无论ARC是否有效，都会调用对象的`dealloc`方法。ARC有效时`dealloc`方法使用参照[第31条：在dealloc方法中只释放引用并解除监听](https://github.com/HelloiWorld/MyLibrary/blob/master/notes/读《Effective%20Objective-C%202.0：编写高质量iOS与OS%20X代码的52个有效方法》有感.md)；此外ARC无效时`dealloc`方法内还需调用`[super dealloc]`方法
* 使用`@autoreleasepool`块替代`NSAutoreleasePool`：ARC下`NSAutoreleasePool`不可用
* 不能使用区域（`NSZone`）：运行时系统中的内存管理本身已极具效率，使用区域来管理内存反而会引起内存使用效率低下以及源代码复杂化等问题。已被忽略。
* 对象型变量不能作为C语言结构体（`struct/union`）的成员：C语言规约上没有办法管理结构体成员的生存周期。除非使用`__unsafe_unretained`修饰，
* 显式转换`id`和`void *`：使用`__bridge`相关用法，首先要了解[第49条：对自定义其内存管理语义的collection使用无缝桥接](https://github.com/HelloiWorld/MyLibrary/blob/master/notes/读《Effective%20Objective-C%202.0：编写高质量iOS与OS%20X代码的52个有效方法》有感.md)，理解转换后引用计数值的变化。

#### 属性
属性声明的属性  | 所有权修饰符
------------- | -------------
`assign`  | `__unsafe_unretained `修饰符
`copy`  | `__strong`修饰符（但是赋值的是被复制的对象）
`retain`  | `__strong`修饰符
`strong`  | `__strong `修饰符
`unsafe_unretained`  | `__unsafe_unretained `修饰符
`weak`  | `__weak`修饰符
这里值得注意的是`assign`对应的所有权修饰符是`__unsafe_unretained`，我们知道`__unsafe_unretained`修饰符具有对象被废弃后指针不会自动清除的特点，且编译器不会对其进行内存管理，所以`assign`不应该去修饰对象类型，而是修饰“纯量类型”。

#### 数组
* `calloc`函数分配内存时使分配区域初始化为0，例`array = (id __strong*)calloc(entries, sizeof(id));`
* `malloc`函数分配的内存区域不会被初始化为0，可在之后用`memset`等函数将内存填充为0。相关用法`array = (id __strong*)malloc(sizeof(id) * entries);`

## ARC的实现
#### `__strong`修饰符
`objc_autoreleaseReturnValue`与`objc_retainAutoreleasedReturnValue`函数协作，可以不将返回对象注册到`autoreleasepool`中而直接传递，省略注册过程以达到最优化。

#### `__weak`修饰符
* 若附有`__weak`修饰符的变量所引用的对象被废弃，则将nil赋值给该变量。
 * `objc_release`
 * 因为引用计数为0所以执行`dealloc`
 * `_objc_rootDealloc`
 * `object_dispose`
 * `objc_destructInstance`
 * `objc_clear_deallocating`
     * 从weak表中获取废弃对象的地址为键值的记录；
     * 将包含在记录中的所有附有`__weak`修饰符变量的地址，赋值为nil；
     * 从weak表中删除该记录；
     * 从引用计数表中删除废弃对象的地址为键值的记录。
* 使用附有`__weak`修饰符的变量，即是使用注册到`autoreleasepool`中的对象。
 * 不支持`__weak`修饰符的类，其类声明中附加了`__attribute__ ((objc_arc_weak_reference_unavailable))`这一属性，同时定义了`NS_AUTOMATED_REFCOUNT_WEAK_UNAVAILABLE`。
 * NSObject类中`allowsWeakReference/retainWeakReference`方法返回NO的情况也不能使用`__weak`修饰符。

#### `__autoreleasing`修饰符
将对象赋值给附有`__autoreleasing`修饰符的变量等同于ARC无效时调用对象的`autorelease`方法。
 
#### 引用计数
获取引用计数数值的函数：`uintptr_t _objc_rootRetainCount(id obj)`（对于已释放的对象及多线程下，不可完全相信）



# 2. Blocks
## Blocks的概念
#### 定义
带有自动变量（局部变量）的匿名函数。（`lambda`）

#### 语法
	^ [返回值类型] [参数列表] 表达式

#### 截获自动变量值
block语法在执行时，截获（保存）的自动变量的值，是其瞬间值，被保存的值无法在表达式体内被重新赋值，在语法结束后修改也无效。

#### `__block`说明符
使用附有`__block`说明符的自动变量可在Block中赋值，该变量称为`__block`变量。

#### 截获的自动变量
截获的自动变量被重新赋值会产生编译错误，但使用其值却不会有任何问题。

## Blocks的实现
> 将Block语法转换成C++源代码：`clang -rewrite-objc 源代码文件名`

后面内容还是看书上例子讲解吧



# 3. Grand Central Dispatch
略🤣