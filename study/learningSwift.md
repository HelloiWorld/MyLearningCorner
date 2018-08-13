# 迁移Swift不明白知识点记录

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



## 4. Swift中的闭包
函数和闭包都是**引用类型**。

* 根据上下文推断参数和返回值类型
* 从单行表达式闭包中隐式返回（也就是闭包体只有一行代码，可以省略return）
* 可以使用简化参数名，如$0, $1(从0开始，表示第i个参数...)
* 提供了尾随闭包语法(Trailing closure syntax)



## 5. Swift中的泛型
...



## 6. Swift中一些常用关键字
#### final
防止属性和方法被重写

#### `@discardableResult`
取消如果不使用方法返回值的警告

#### `@objc`
允许Objective-C调用

#### `@mutating`
允许在方法内修改实例变量

#### `lazy`
* 懒加载本质上是一个闭包
* 懒加载会在第一次访问的时候执行, 闭包执行结束后, 会把结果保存在属性中，后续调用直接返回 属性的内容
* 懒加载的属性会分配空间, 存储值
* 只要调用过一次, 懒加载后面的闭包再也不会执行了

#### `fallthrough`
swift中switch语句无需在每一个case后加break跳出，但如果希望继续执行就需要加`fallthrough`



## 7. Swift中一些符号和运算符的差异和变化
#### `nil`
OC中`nil`表示的是指针，用于对象；Swift中`nil`表示没有值，如Int和Bool没有值的时候不再默认是0和false

#### `==` 与 `===` 
`==`只比较内容；`===`不仅比较内容还会比较内存地址



## 8. Swift中写法上的一些改变
#### 单例写法
```
class Singleton {
  static let sharedInstance = Singleton()
}
```
或者

```
class Singleton {
  static let sharedInstance: Singleton = {
      let instance = Singleton()
      // setup code
      return instance
  }()
}
```

#### 耗时操作
```
DispatchQueue.global(qos: .default).async {
	//处理耗时操作
   DispatchQueue.main.async {
       //操作完成，调用主线程来刷新界面
   }
}
```

#### 延时执行
```
DispatchQueue.main.asyncAfter(deadline: DispatchTime.now()+0.3) {
	// do some thing ...
}
```

 