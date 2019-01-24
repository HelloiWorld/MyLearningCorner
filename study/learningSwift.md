# 迁移Swift知识点记录

## 1. Swift 4.0 中的 open，public，internal，fileprivate，private
#### private 
private访问级别所修饰的属性或者方法只能在当前类里访问。

#### fileprivate 
fileprivate访问级别所修饰的属性或者方法在当前的Swift源文件里可以访问。

#### internal（默认访问级别，internal修饰符可写可不写） 
internal访问级别所修饰的属性或方法在源代码所在的整个模块都可以访问。 
如果是框架或者库代码，则在整个框架内部都可以访问，框架由外部代码所引用时，则不可以访问。 
如果是App代码，也是在整个App代码，也是在整个App内部可以访问。

#### public 
可以被任何人访问。但其他module中不可以被override和继承，而在module内可以被override和继承。

#### open 
可以被任何人使用，包括override和继承。

### 访问顺序
现在的访问权限则依次为：open，public，internal，fileprivate，private。


## 2. Class和Struct的区别？
* Swift中，Class是引用类型，Struct和Enum是值类型。引用类型在传递的时候，默认是同一个内存对象的引用，类似strong。而值类型在传递的时候，都是copy。
* Class可以继承，Struct不能继承。
* Class分配在堆中，Struct一般分配在栈中。

### 值类型
* 值类型是不可变状态，在传递的时候就固定了，不需要考虑会在别处被修改。
* 并不是每次都会copy，当值不发生改变的时候不会进行此拷贝操作。

Struct性能比Class更好：
 
* 通常Struct在栈上分配内存，Class在堆上分配内存。在堆上分配内存的性能是要低于栈上的，因为堆上不得不做很多锁之类的操作来保证多线程的时候不会有问题。
* Struct无需引用计数，而Classs需要。引用计数会造成隐式的额外开销
* Struct不能继承，所有在方法执行的时候，是static dispatch；而Class是dynacmic dispatch，意味着Class需要通过virtual table来动态的找到执行的方法。 


## 3. Swift中协议相比OC有什么变化？
* Swift中协议是一种类型
* Swift中协议可以继承
* Swift中协议可以（条件）拓展
* **Swift中协议可以有自己的默认实现**，即直接完成所要做的事，外部只需调起即可


## 4. Swift中的闭包
函数和闭包都是**引用类型**（也就是说对其赋值对象的操作也会影响到原来的对象）。

* 根据上下文推断参数和返回值类型
* 从单行表达式闭包中隐式返回（也就是闭包体只有一行代码，可以省略return）
* 可以使用简化参数名，如$0, $1(从0开始，表示第i个参数...)
* 提供了尾随闭包语法(Trailing closure syntax)


## 5. Swift中的泛型
利用泛型设置有效值，限定区间

```
func setEffectiveValue<T: Comparable>(value: inout T, min: T?, max: T?) {
    if let min = min {
        value = value >= min ? value : min
    }
    if let max = max {
        value = value < max ? value : max
    }
}
```


## 6. Swift中一些符号和运算符的差异和变化
#### `nil`
OC中`nil`表示的是指针，用于对象；Swift中`nil`表示没有值，如Int和Bool没有值的时候不再默认是0和false

#### `switch`
swift中switch-case支持枚举，相应的也支持基本数据类型，在每个case中不再需要添加`break`标明语句结束，而想要继续执行下一个case需要增加`fallthrough`关键字

#### `guard`
swift中特有的`guard`语句是对`if xx { return }`写法的补充，通常用法是在语句开始前利用`guard`对所有可选类型校验值是否存在并解包

#### 枚举
swift中枚举支持默认值是非Int类型，也可以嵌套，另外还有很多高级用法请查阅相关资料

```
enum PlatformName: String {
    case wechat_session = "微信好友"
    case wechat_timeline = "朋友圈"
    case qq = "QQ好友"
    case qzone = "QQ空间"
}
```
或是

```
enum PlatformType {
	 enum wechat: Int {
	     case session = 1
	     case timeline = 2
	 }
    enum qq: Int {
    	  case session = 1
    	  case qzone = 2
    }
}
```

#### `==` 与 `===` 
`==`只比较内容；`===`不仅比较内容还会比较内存地址

 