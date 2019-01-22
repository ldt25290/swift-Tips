# SwiftTips (๑•̀ㅂ•́)و✧
记录iOS开发中的一些知识点  

[![Language: Swift 4.2](https://img.shields.io/badge/language-swift4.2-f48041.svg?style=flat)](https://developer.apple.com/swift)
![Platform: iOS 12](https://img.shields.io/badge/platform-iOS-blue.svg?style=flat)


[1.常用的几个高阶函数](#1)  
[2.高阶函数扩展](#2)  
[3.优雅的判断多个值中是否包含某一个值](#3)  
[4.Hashable、Equatable和Comparable协议](#4)  
[5.可变参数函数](#5)  
[6.where关键字](#6)  
[7.switch中判断枚举类型,尽量避免使用default](#7)  
[8.iOS9之后全局动态修改StatusBar样式](#8)  
[9.使用面向协议实现app的主题功能](#9)  
[10.swift中多继承的实现](#10)  
[11.华丽的TableView刷新动效](#11)  
[12.实现一个不基于Runtime的KVO](#12)  
[13.实现多重代理](#13)   
[14.自动检查控制器是否被销毁](#14)  
[15.向控制器中注入代码](#15)  
[16.给Extension添加存储属性](#16)  
[17.用闭包实现按钮的链式点击事件](#17)  
[18.用闭包实现手势的链式监听事件](#18)  
[19.用闭包实现通知的监听事件](#19)  
[20.AppDelegate解耦](#20)  
[21.常见的编译器诊断指令](#21)  
[22.最后执行的defer代码块](#22)  
[23.定义全局常量](#23)  
[24.使用Codable协议解析JSON](#24)  
[25.dispatch_once替代方案](#25)  
[26.被废弃的+load()和+initialize()](#26)  
[27.交换方法 Method Swizzling](#27)    
[28.获取View的指定子视图](#28)  
[29.线程安全: 互斥锁和自旋锁(10种)](#29)  




<h2 id="1">1.常用的几个高阶函数</h2>  

函数式编程在swift中有着广泛的应用,下面列出了几个常用的高阶函数.
#### 1. sorted
常用来对数组进行排序.顺便感受下函数式编程的多种姿势.

##### 1. 使用sort进行排序,不省略任何类型

```swift
let intArr = [13, 45, 27, 80, 22, 53]

let sortOneArr = intArr.sorted { (a: Int, b: Int) -> Bool in
    return a < b
}
// [13, 22, 27, 45, 53, 80]
```
##### 2. 编译器可以自动推断出返回类型,所以可以省略

```swift
let sortTwoArr = intArr.sorted { (a: Int, b: Int) in
    return a < b
}
// [13, 22, 27, 45, 53, 80]
```

##### 3. 编译器可以自动推断出参数类型,所以可以省略

```swift
let sortThreeArr = intArr.sorted { (a, b) in
    return a < b
}
// [13, 22, 27, 45, 53, 80]
```

##### 4. 编译器可以自动推断出参数个数,所以可以用$0,$1替代

```swift
let sortFourArr = intArr.sorted {
    return $0 < $1
}
// [13, 22, 27, 45, 53, 80]
```

##### 5. 如果闭包中的函数体只有一行,且需要有返回值,return可以省略

```swift
let sortFiveArr = intArr.sorted {
    $0 < $1
}
// [13, 22, 27, 45, 53, 80]
```

##### 6. 最简化: 可以直接传入函数`<`

```swift
let sortSixArr = intArr.sorted(by: <)
// [13, 22, 27, 45, 53, 80]
```

#### 2. map和compactMap
##### 1. map: 对数组中每个元素做一次处理.

```swift
let mapArr = intArr.map { $0 * $0 }
// [169, 2025, 729, 6400, 484, 2809]
```
##### 2. compactMap: 和map类似,但可以过滤掉nil,还可以对可选类型进行解包.

```swift
let optionalArr = [nil, 4, 12, 7, Optional(3), 9]
let compactMapArr = optionalArr.compactMap { $0 }
// [4, 12, 7, 3, 9]
```

#### 3. filter: 将符合条件的元素重新组合成一个数组

```swift
let evenArr = intArr.filter { $0 % 2 == 0 }
// [80, 22]
```

#### 4. reduce: 将数组中的元素合并成一个

```swift
// 组合成一个字符串
let stringArr = ["1", "2", "3", "*", "a"]
let allStr = stringArr.reduce("") { $0 + $1 }
// 123*a

// 求和
let sum = intArr.reduce(0) { $0 + $1 }
// 240
```
#### 5. 高阶函数可以进行链式调用.比如,求一个数组中偶数的平方和

```swift
let chainArr = [4, 3, 5, 8, 6, 2, 4, 7]

let resultArr = chainArr.filter {
                            $0 % 2 == 0
                        }.map {
                            $0 * $0
                        }.reduce(0) {
                            $0 + $1
                        }
// 136
```


<h2 id="2">2.高阶函数扩展</h2> 


#### 1. map函数的实现原理

```swift
extension Sequence {
    
    // 可以将一些公共功能注释为@inlinable,给编译器提供优化跨模块边界的泛型代码的选项
    @inlinable
    public func customMap<T>(
        _ transform: (Element) throws -> T
        ) rethrows -> [T] {
        let initialCapacity = underestimatedCount
        var result = ContiguousArray<T>()
        
        // 因为知道当前元素个数,所以一次性为数组申请完内存,避免重复申请
        result.reserveCapacity(initialCapacity)
        
        // 获取所有元素
        var iterator = self.makeIterator()
        
        // 将元素通过参数函数处理后添加到数组中
        for _ in 0..<initialCapacity {
            result.append(try transform(iterator.next()!))
        }
        // 如果还有剩下的元素,添加进去
        while let element = iterator.next() {
            result.append(try transform(element))
        }
        return Array(result)
    }
}

```
map的实现无非就是创建一个空数组,通过for循环遍历将每个元素通过传入的函数处理后添加到空数组中,只不过swift的实现更加高效一点.

关于其余相关高阶函数的实现:[Sequence.swift
](https://github.com/apple/swift/blob/swift-4.2-branch/stdlib/public/core/Sequence.swift)

#### 2. 关于数组中用到的其他的一些高阶函数

```swift
class Pet {
    let type: String
    let age: Int
    
    init(type: String, age: Int) {
        self.type = type
        self.age = age
    }
}

var pets = [
            Pet(type: "dog", age: 5),
            Pet(type: "cat", age: 3),
            Pet(type: "sheep", age: 1),
            Pet(type: "pig", age: 2),
            Pet(type: "cat", age: 3),
            ]
```
###### 1. 遍历所有元素
```swift         
pets.forEach { p in
   print(p.type)
}
```
###### 2. 是否包含满足条件的元素
```swift
let cc = pets.contains { $0.type == "cat" }
```
###### 3. 第一次出现满足条件的元素的位置
```swift
let firstIndex = pets.firstIndex { $0.age == 3 }
// 1
```
###### 4. 最后一次出现满足条件的元素的位置
```swift
let lastIndex = pets.lastIndex { $0.age == 3 }
// 4
```
###### 5. 根据年龄从大到小进行排序
```swift
let sortArr = pets.sorted { $0.age < $1.age }
```
###### 6. 获取age大于3的元素
```swift
let arr1 = pets.prefix { $0.age > 3 }
// [{type "dog", age 5}]
```
###### 7. 获取age大于3的取反的元素
```swift
let arr2 = pets.drop { $0.age > 3 }
// [{type "cat", age 3}, {type "sheep", age 1}, {type "pig", age 2}, {type "cat", age 3}]

```
###### 8. 将字符串转化为数组
```swift
let line = "BLANCHE:   I don't want realism. I want magic!"

let wordArr = line.split(whereSeparator: { $0 == " " })
// ["BLANCHE:", "I", "don\'t", "want", "realism.", "I", "want", "magic!"]
```

<h2 id="3">3.优雅的判断多个值中是否包含某一个值</h2>  

我们最常用的方式

```swift
let string = "One"

if string == "One" || string == "Two" || string == "Three" {
    print("One")
}
```
这种方式是可以,但可阅读性不够,那有啥好的方式呢?  
#### 1. 我们可以利用`contains`:

```swift
if ["One", "Two", "Three"].contains(where: { $0 == "One"}) {
	print("One")
}
```
#### 2. 自己手动实现一个`any`  

##### 使用:
```swift

if string == any(of: "One", "Two", "Three") {
    print("One")
}

```

##### 实现:

```swift
func any<T: Equatable>(of values: T...) -> EquatableValueSequence<T> {
    return EquatableValueSequence(values: values)
}

struct EquatableValueSequence<T: Equatable> {
    static func ==(lhs: EquatableValueSequence<T>, rhs: T) -> Bool {
        return lhs.values.contains(rhs)
    }
    
    static func ==(lhs: T, rhs: EquatableValueSequence<T>) -> Bool {
        return rhs == lhs
    }
    
    fileprivate let values: [T]
}
```
这样做的前提是any中传入的值需要实现`Equatable`协议.


<h2 id="4">4. Hashable、Equatable和Comparable协议</h2>  

#### 1. Hashable  
实现Hashable协议的方法后我们可以根据`hashValue`方法来获取该对象的哈希值.  
字典中的value的存储就是根据key的`hashValue`,所以所有字典中的key都要实现Hashable协议.

```swift
class Animal: Hashable {
    
    var hashValue: Int {
        return self.type.hashValue ^ self.age.hashValue
    }
    
    let type: String
    let age: Int
    
    init(type: String, age: Int) {
        self.type = type
        self.age = age
    }
}

let a1 = Animal(type: "Cat", age: 3)
a1.hashValue
// 哈希值
```
#### 2. Equatable协议
实现Equatable协议后,就可以用`==`符号来判断两个对象是否相等了.

```swift
class Animal: Equatable, Hashable {
    
    static func == (lhs: Animal, rhs: Animal) -> Bool {
        if lhs.type == rhs.type && lhs.age == rhs.age{
            return true
        }else {
            return false
        }
    }
        
    let type: String
    let age: Int
    
    init(type: String, age: Int) {
        self.type = type
        self.age = age
    }
}

let a1 = Animal(type: "Cat", age: 3)
let a2 = Animal(type: "Cat", age: 4)

a1 == a2
// false
```
#### 3. Comparable协议
基于Equatable基础上的Comparable类型,实现相关的方法后可以使用`<`、`<=`、`>=`、`>` 等符号进行比较.

```swift
class Animal: Comparable {
    // 只根据年龄选项判断
    static func < (lhs: Animal, rhs: Animal) -> Bool {
        if lhs.age < rhs.age{
            return true
        }else {
            return false
        }
    }
    
    let type: String
    let age: Int
    
    init(type: String, age: Int) {
        self.type = type
        self.age = age
    }
}

let a1 = Animal(type: "Cat", age: 3)
let a2 = Animal(type: "Cat", age: 4)
let a3 = Animal(type: "Cat", age: 1)
let a4 = Animal(type: "Cat", age: 6)

// 按照年龄从大到小排序
let sortedAnimals = [a1, a2, a3, a4].sorted(by: <)
```
在日常开发中会涉及到大量对自定义对象的比较操作,所以`Comparable`协议的用途还是比较广泛的.

`Comparable`协议除了应用在类上,还可以用在**结构体**和**枚举**上.

 <h2 id="5">5.可变参数函数</h2>  
在定义函数的时候,如果参数的个数不确定时,需要使用可变参数函数.举个例子,对数组的求和.

```swift
// 常用的姿势
[2, 3, 4, 5, 6, 7, 8, 9].reduce(0) { $0 + $1 }
// 44

// 使用可变参数函数
sum(values: 2, 3, 4, 5, 6, 7, 8, 9)
// 44

// 可变参数的类型是个数组
func sum(values:Int...) -> Int {
    var result = 0
    values.forEach({ a in
        result += a
    })
    return result
}

```

应用: 

```swift

// 给UIView添加子控件
let view = UIView()
let label = UILabel()
let button = UIButton()
view.add(view, label, button)

extension UIView {
    /// 同时添加多个子控件
    ///
    /// - Parameter subviews: 单个或多个子控件
    func add(_ subviews: UIView...) {
        subviews.forEach(addSubview)
    }
}

```

<h2 id="6">6.where关键字</h2>   

where的主要作用是用来做限定.  

#### 1. `for`循环的时候用来做条件判断

```swift
// 只遍历数组中的偶数
let arr = [11, 12, 13, 14, 15, 16, 17, 18]
for num in arr where num % 2 == 0 {
    // 12 14 16 18
}
```

#### 2. 在`try` `catch`的时候做条件判断

```swift
enum ExceptionError:Error{
    case httpCode(Int)
}

func throwError() throws {
    throw ExceptionError.httpCode(500)
}

do{
    try throwError()
// 通过where添加限定条件
}catch ExceptionError.httpCode(let httpCode) where httpCode >= 500{
    print("server error")
}catch {
    print("other error")
}
```  

#### 3. switch语句做限定条件

```swift
let student:(name:String, score:Int) = ("小明", 59)
switch student {
case let (_,score) where score < 60:
    print("不及格")
default:
    print("及格")
}
```  

#### 4. 限定泛型需要遵守的协议

```swift
//第一种写法
func genericFunctionA<S>(str:S) where S:ExpressibleByStringLiteral{
    print(str)
}
//第二种写法
func genericFunctionB<S:ExpressibleByStringLiteral>(str:S){
    print(str)
}
```
#### 5. 为指定的类添加对应的协议扩展

```swift
// 为Numeric在Sequence中添加一个求和扩展方法
extension Sequence where Element: Numeric {
    var sum: Element {
        var result: Element = 0
        for item  in self {
            result += item
        }
        return result
    }
}

print([1,2,3,4].sum) // 10
```

#### 6. 为某些高阶函数的限定条件

```swift
let names = ["Joan", "John", "Jack"]
let firstJname = names.first(where: { (name) -> Bool in
    return name.first == "J"
})
// "Joan"
let fruits = ["Banana", "Apple", "Kiwi"]
let containsBanana = fruits.contains(where: { (fruit) in
    return fruit == "Banana"
})
// true
```

参考: [Swift where 关键字](https://www.jianshu.com/p/1546594b856b)  


<h2 id="7">7.switch中判断枚举类型,尽量避免使用default</h2> 

通过`switch`语句来判断枚举类型,不使用`default`,如果后期添加新的枚举类型,而忘记在`switch`中处理,会报错,这样可以提高代码的健壮性.

```swift
enum State {        
    case loggedIn
    case loggedOut
    case startUI
}
    
func handle(_ state: State) {
    switch state {
    case .loggedIn:
         showMainUI()
    case .loggedOut:
        showLoginUI()
        
        // Compiler error: Switch must be exhaustive
    }
}

```


 <h2 id="8">8.iOS9之后全局动态修改StatusBar样式</h2>  
 
#### 1. 局部修改StatusBar样式
最常用的方法是通过控制器来修改`StatusBar`样式

```swift
override var preferredStatusBarStyle: UIStatusBarStyle {
    return .lightContent
}
```
注意:如果当前控制器有导航控制器,需要在导航控制器中这样设置(如下代码),不然不起作用.

```swift
override var preferredStatusBarStyle: UIStatusBarStyle {
    return topViewController?.preferredStatusBarStyle ?? .default
}
```
这样做的好处是,可以针对不同的控制器设置不同的`StatusBar`样式,但有时往往会多此一举,略嫌麻烦,那如何全局统一处理呢?

#### 2. iOS9之前全局修改StatusBar样式
iOS9之前的做法比较简单,在`plist`文件中设置`View controller-based status bar appearance`为`NO`.  

在需要设置的地方添加

```swift
UIApplication.shared.setStatusBarStyle(.default, animated: true)
```
这样全局设置`StatusBar`样式就可以了,但iOS9之后`setStatusBarStyle`方法被废弃了,苹果推荐使用`preferredStatusBarStyle`,也就是上面那种方法.
#### 3. iOS9之后全局修改StatusBar样式
我们可以用`UIAppearance`和导航栏的`barStyle`去全局设置`StatusBar`的样式.

- `UIAppearance`属性可以做到全局修改样式.
- 导航栏的`barStyle`决定了`NavigationBar`的外观,而`barStyle`属性改变会联动到`StatusBar`的样式.
	1. 当`barStyle = .default`,表示导航栏的为默认样式,`StatusBar`的样式为了和导航栏区分,就会变成**黑色**.
	2. 当`barStyle = .black`,表示导航栏的颜色为深黑色,`StatusBar`的样式为了和导航栏区分,就会变成**白色**.

	这个有点绕,总之就是`StatusBar`的样式和导航栏的样式反着来.
	
具体实现:

```swift
@IBAction func segmentedControl(_ sender: UISegmentedControl) {
        
    switch sender.selectedSegmentIndex {
    case 0:
    	 // StatusBar为黑色,导航栏颜色为白色
        UINavigationBar.appearance().barStyle = .default
        UINavigationBar.appearance().barTintColor = UIColor.white
    default:
    	 // StatusBar为白色,导航栏颜色为深色
        UINavigationBar.appearance().barStyle = .black
        UINavigationBar.appearance().barTintColor = UIColor.darkNight
    }
    
    // 刷新window下的子控件
    UIApplication.shared.windows.forEach {
        $0.reload()
    }
}

public extension UIWindow {
    func reload() {
        subviews.forEach { view in
            view.removeFromSuperview()
            addSubview(view)
        }
    }
}
```

<img src="http://pcb5zz9k5.bkt.clouddn.com/changeStatusBarStyle2.gif" width=250>


<h2 id="9">9.使用面向协议实现app的主题功能</h2>

#### 1. `UIAppearance`修改全局样式

做为修改全局样式的`UIAppearance`用起来还是很方便的,比如要修改所有`UILabel`的文字颜色.

```swift
UILabel.appearance().textColor = labelColor
```
又或者我们只想修改某个`CustomView`层级下的子控件`UILabel`

```swift
UILabel.appearance(whenContainedInInstancesOf: [CustomView.self]).textColor = labelColor
```

#### 2. 主题协议,以及实现  

定义好协议中需要实现的属性和方法

```swift
protocol Theme {
    
    // 自定义的颜色
    var tint: UIColor { get }
    // 定义导航栏的样式,为了联动状态栏(具体见第9小点)
    var barStyle: UIBarStyle { get }
    
    var labelColor: UIColor { get }
    var labelSelectedColor: UIColor { get }
    
    var backgroundColor: UIColor { get }
    var separatorColor: UIColor { get }
    var selectedColor: UIColor { get }
    
    // 设置主题样式
    func apply(for application: UIApplication)
    
    // 对特定主题样式进行扩展
    func extend()
}
```
对协议添加`extension`,这样做的好处是,如果有多个结构体或类实现了协议,而每个结构体或类需要实现相同的方法,这些方法就可以统一放到`extension`中处理,大大提高了代码的复用率.  
如果结构体或类有着相同的方法实现,那么结构体或类的实现会**覆盖**掉协议的`extension`中的实现.

```swift
extension Theme {
    
    func apply(for application: UIApplication) {
        application.keyWindow?.tintColor = tint
        
        
        UITabBar.appearance().with {
            $0.barTintColor = tint
            $0.tintColor = labelColor
        }
        
        UITabBarItem.appearance().with {
            $0.setTitleTextAttributes([.foregroundColor : labelColor], for: .normal)
            $0.setTitleTextAttributes([.foregroundColor : labelSelectedColor], for: .selected)
        }
        

        UINavigationBar.appearance().with {
            $0.barStyle = barStyle
            $0.tintColor = tint
            $0.barTintColor = tint
            $0.titleTextAttributes = [.foregroundColor : labelColor]
        }
        
       UITextView.appearance().with {
            $0.backgroundColor = selectedColor
            $0.tintColor = tint
            $0.textColor = labelColor
        }
        
        extend()
        
        application.windows.forEach { $0.reload() }
    }
    
    // ... 其余相关UIAppearance的设置
    
    
    // 如果某些属性需要在某些主题下定制,可在遵守协议的类或结构体下重写
    func extend() {
        // 在主题中实现相关定制功能
    }
}

```

#### 3. 对主题某些样式的自定义
Demo中白色主题的`UISegmentedControl`需要设置特定的颜色,我们可以在`LightTheme`的`extension`中重写`extend()`方法.

```swift
extension LightTheme {
    
    // 需要自定义的部分写在这边
    func extend() {
        UISegmentedControl.appearance().with {
            $0.tintColor = UIColor.darkText
            $0.setTitleTextAttributes([.foregroundColor : labelColor], for: .normal)
            $0.setTitleTextAttributes([.foregroundColor : UIColor.white], for: .selected)
        }
        UISlider.appearance().tintColor = UIColor.darkText
    }
}

```

#### 4. 主题切换
在设置完`UIAppearance`后需要对所有的控件进行刷新,这个操作放在`apply`方法中.具体实现

```swift
public extension UIWindow {
    /// 刷新所有子控件
    func reload() {
        subviews.forEach { view in
            view.removeFromSuperview()
            addSubview(view)
        }
    }
}
```

[示例Demo](https://github.com/DarielChen/SwiftTips/tree/master/Demo/9.%E4%BD%BF%E7%94%A8%E9%9D%A2%E5%90%91%E5%8D%8F%E8%AE%AE%E5%AE%9E%E7%8E%B0app%E7%9A%84%E4%B8%BB%E9%A2%98%E5%8A%9F%E8%83%BD)  
[实现效果](http://pcb5zz9k5.bkt.clouddn.com/themeDemo.gif)


<h2 id="10">10.swift中多继承的实现</h2>  

#### 1. 实现过程
swift本身并不支持多继承,但我们可以根据已有的API去实现. 
 
swift中的类可以遵守多个协议,但是只可以继承一个类,而值类型(结构体和枚举)只能遵守单个或多个协议,不能做继承操作.  

多继承的实现:**协议的方法可以在该协议的`extension`中实现**

```swift
protocol Behavior {
    func run()
}
extension Behavior {
    func run() {
        print("Running...")
    }
}

struct Dog: Behavior {}

let myDog = Dog()
myDog.run() // Running...
```
无论是结构体还是类还是枚举都可以遵守多个协议,所以多继承就这么做到了.

#### 2. 通过多继承为`UIView`扩展方法

```swift
// MARK: - 闪烁功能
protocol Blinkable {
    func blink()
}
extension Blinkable where Self: UIView {
    func blink() {
        alpha = 1
        
        UIView.animate(
            withDuration: 0.5,
            delay: 0.25,
            options: [.repeat, .autoreverse],
            animations: {
                self.alpha = 0
        })
    }
}

// MARK: - 放大和缩小
protocol Scalable {
    func scale()
}
extension Scalable where Self: UIView {
    func scale() {
        transform = .identity
        
        UIView.animate(
            withDuration: 0.5,
            delay: 0.25,
            options: [.repeat, .autoreverse],
            animations: {
                self.transform = CGAffineTransform(scaleX: 1.5, y: 1.5)
        })
    }
}

// MARK: - 添加圆角
protocol CornersRoundable {
    func roundCorners()
}
extension CornersRoundable where Self: UIView {
    func roundCorners() {
        layer.cornerRadius = bounds.width * 0.1
        layer.masksToBounds = true
    }
}

extension UIView: Scalable, Blinkable, CornersRoundable {}

 cyanView.blink()
 cyanView.scale()
 cyanView.roundCorners()
```

<img src="http://pcb5zz9k5.bkt.clouddn.com/blink_scale_corner.gif" width=250>


#### 3. 多继承钻石问题(Diamond Problem),及解决办法
请看下面代码

```swift
protocol ProtocolA {
    func method()
}

extension ProtocolA {
    func method() {
        print("Method from ProtocolA")
    }
}

protocol ProtocolB {
    func method()
}

extension ProtocolB {
    func method() {
        print("Method from ProtocolB")
    }
}

class MyClass: ProtocolA, ProtocolB {}
```
此时`ProtocolA`和`ProtocolB`都有一个默认的实现方法`method()`,由于编译器不知道继承过来的`method()`方法是哪个,就会报错.
> 💎钻石问题,当某一个类或值类型在继承图谱中有多条路径时就会发生.

解决方法:  
	1. 在目标值类型或类中重写那个发生冲突的方法`method()`.  
	2. 直接修改协议中重复的方法

相对来时第二种方法会好一点,所以多继承要注意,尽量避免多继承的协议中的方法的重复.


<h2 id="11">11.华丽的TableView刷新动效</h2>  

[先看效果](http://pcb5zz9k5.bkt.clouddn.com/TableViewRefreshAnimation2.gif
)(由于这个页面的内容有点多,我尽量不放加载比较耗时的文件)

#### 1. 简单的实现
我们都知道`TableView`的刷新动效是设置在`tableView(_:,willDisplay:,forRowAt:)`这个方法中的.

```swift
override func tableView(_ tableView: UITableView, willDisplay cell: UITableViewCell, forRowAt indexPath: IndexPath) {
    
    cell.alpha = 0
    UIView.animate(
        withDuration: 0.5,
        delay: 0.05 * Double(indexPath.row),
        animations: {
            cell.alpha = 1
    })
}      
```
这样一个简单的淡入效果就OK了.但这样做显然不够优雅,我们如果要在多个`TableView`使用这个效果该怎样封装呢?
#### 2. 使用工厂设计模式进行封装

##### 1. creator(创建者): `Animator`,用来传入参数,和设置动画

```swift
// Animation接收三个参数
typealias Animation = (UITableViewCell, IndexPath, UITableView) -> Void

final class Animator {
    private var hasAnimatedAllCells = false
    private let animation: Animation

    init(animation: @escaping Animation) {
        self.animation = animation
    }

    func animate(cell: UITableViewCell, at indexPath: IndexPath, in tableView: UITableView) {
        guard !hasAnimatedAllCells else {
            return
        }
        animation(cell, indexPath, tableView)
		// 确保每个cell动画只执行一次
        hasAnimatedAllCells = tableView.isLastVisibleCell(at: indexPath)
    }
}
```

##### 2. product（产品): `AnimationFactory`,用来设置不同的动画类型

```swift
enum AnimationFactory {
    
    static func makeFade(duration: TimeInterval, delayFactor: Double) -> Animation {
        return { cell, indexPath, _ in
            cell.alpha = 0
            
            UIView.animate(
                withDuration: duration,
                delay: delayFactor * Double(indexPath.row),
                animations: {
                    cell.alpha = 1
            })
        }
    }
    
    // ... 
}
```
将所有的动画设置封装在`Animation`的闭包中.  


最后我们就可以在`tableView(_:,willDisplay:,forRowAt:)`这个方法中使用了

```swift
let animation = AnimationFactory.makeFade(duration: 0.5, delayFactor: 0.05)
let animator = TableViewAnimator(animation: animation)
animator.animate(cell: cell, at: indexPath, in: tableView)
```

动画相关的可以参考我之前写的文章  [猛击](https://www.jianshu.com/p/6af8a7a8a15a)  
[实现效果](http://pcb5zz9k5.bkt.clouddn.com/TableViewRefreshAnimation2.gif)  
[示例Demo](https://github.com/DarielChen/SwiftTips/tree/master/Demo/11.%E5%8D%8E%E4%B8%BD%E7%9A%84TableView%E5%88%B7%E6%96%B0%E5%8A%A8%E6%95%88)  



<h2 id="12">12.实现一个不基于Runtime的KVO</h2> 
 
Swift并没有在语言层级上支持KVO,如果要使用必须导入`Foundation`框架, 被观察对象必须继承自`NSObject`,这种实现方式显然不够优雅.  

KVO本质上还是通过拿到属性的set方法去搞事情,基于这样的原理我们可以自己去实现.

#### 1. 实现
话不多说,直接贴代码,新建一个`Observable`文件

```swift
public class Observable<Type> {
    
    // MARK: - Callback
    fileprivate class Callback {
        fileprivate weak var observer: AnyObject?
        fileprivate let options: [ObservableOptions]
        fileprivate let closure: (Type, ObservableOptions) -> Void
        
        fileprivate init(
            observer: AnyObject,
            options: [ObservableOptions],
            closure: @escaping (Type, ObservableOptions) -> Void) {
            
            self.observer = observer
            self.options = options
            self.closure = closure
        }
    }
    
    // MARK: - Properties
    public var value: Type {
        didSet {
            removeNilObserverCallbacks()
            notifyCallbacks(value: oldValue, option: .old)
            notifyCallbacks(value: value, option: .new)
        }
    }
    
    private func removeNilObserverCallbacks() {
        callbacks = callbacks.filter { $0.observer != nil }
    }
    
    private func notifyCallbacks(value: Type, option: ObservableOptions) {
        let callbacksToNotify = callbacks.filter { $0.options.contains(option) }
        callbacksToNotify.forEach { $0.closure(value, option) }
    }
    
    // MARK: - Object Lifecycle
    public init(_ value: Type) {
        self.value = value
    }
    
    // MARK: - Managing Observers
    private var callbacks: [Callback] = []
    
    
    /// 添加观察者
    ///
    /// - Parameters:
    ///   - observer: 观察者
    ///   - removeIfExists: 如果观察者存在需要移除
    ///   - options: 被观察者
    ///   - closure: 回调
    public func addObserver(
        _ observer: AnyObject,
        removeIfExists: Bool = true,
        options: [ObservableOptions] = [.new],
        closure: @escaping (Type, ObservableOptions) -> Void) {
        
        if removeIfExists {
            removeObserver(observer)
        }
        
        let callback = Callback(observer: observer, options: options, closure: closure)
        callbacks.append(callback)
        
        if options.contains(.initial) {
            closure(value, .initial)
        }
    }
    
    public func removeObserver(_ observer: AnyObject) {
        callbacks = callbacks.filter { $0.observer !== observer }
    }
}

// MARK: - ObservableOptions
public struct ObservableOptions: OptionSet {
    
    public static let initial = ObservableOptions(rawValue: 1 << 0)
    public static let old = ObservableOptions(rawValue: 1 << 1)
    public static let new = ObservableOptions(rawValue: 1 << 2)
    
    public var rawValue: Int
    
    public init(rawValue: Int) {
        self.rawValue = rawValue
    }
}

```
使用起来和KVO差不多.  
#### 2. 使用
需要监听的类

```swift
public class User {
    // 监听的属性需要是Observable类型
    public let name: Observable<String>
    
    public init(name: String) {
        self.name = Observable(name)
    }
}
```
使用

```swift
// 创建对象
let user = User(name: "Made")

// 设置监听
user.name.addObserver(self, options: [.new]) { name, change in
    print("name:\(name), change:\(change)")
}

// 修改对象的属性
user.name.value = "Amel"  // 这时就可以被监听到

// 移除监听
user.name.removeObserver(self)

```
> 注意: 在使用过程中,如果改变value, addObserver方法不调用,很有可能是Observer对象已经被释放掉了.

<h2 id="13">13.实现多重代理</h2>  

作为iOS开发中最常用的设计模式之一`Delegate`,只能是一对一的关系,如果要一对多,就只能使用`NSNotification`了,但我们可以有更好的解决方案,多重代理.

#### 1. 多重代理的实现过程

##### 1. 定义协议

```swift
protocol MasterOrderDelegate: class {
    func toEat(_ food: String)
}
```

##### 2. 定义一个类: 用来管理遵守协议的类

这边用了`NSHashTable`来存储遵守协议的类,`NSHashTable`和`NSSet`类似,但又有所不同,总的来说有这几个特点:  
	1. `NSHashTable`中的元素可以通过`Hashable`协议来判断是否相等.  
	2. `NSHashTable`中的元素如果是弱引用,对象销毁后会被移除,可以避免循环引用.  

```swift
class masterOrderDelegateManager : MasterOrderDelegate {
    private let multiDelegate: NSHashTable<AnyObject> = NSHashTable.weakObjects()

    init(_ delegates: [MasterOrderDelegate]) {
        delegates.forEach(multiDelegate.add)
    }
    
    // 协议中的方法,可以有多个
    func toEat(_ food: String) {
        invoke { $0.toEat(food) }
    }
    
    // 添加遵守协议的类
    func add(_ delegate: MasterOrderDelegate) {
        multiDelegate.add(delegate)
    }
    
    // 删除指定遵守协议的类
    func remove(_ delegateToRemove: MasterOrderDelegate) {
        invoke {
            if $0 === delegateToRemove as AnyObject {
                multiDelegate.remove($0)
            }
        }
    }
    
    // 删除所有遵守协议的类
    func removeAll() {
        multiDelegate.removeAllObjects()
    }

    // 遍历所有遵守协议的类
    private func invoke(_ invocation: (MasterOrderDelegate) -> Void) {
        for delegate in multiDelegate.allObjects.reversed() {
            invocation(delegate as! MasterOrderDelegate)
        }
    }
}
```

##### 3. 其余部分

```swift
class Master {
    weak var delegate: MasterOrderDelegate?
    
    func orderToEat() {
        delegate?.toEat("meat")
    }
}

class Dog {
}
extension Dog: MasterOrderDelegate {
    func toEat(_ food: String) {
        print("\(type(of: self)) is eating \(food)")
    }
    
}

class Cat {
}
extension Cat: MasterOrderDelegate {
    func toEat(_ food: String) {
        print("\(type(of: self)) is eating \(food)")
    }
}

let cat = Cat()
let dog = Dog()
let cat1 = Cat()

let master = Master()
// master的delegate是弱引用,所以不能直接赋值
let delegate = masterOrderDelegateManager([cat, dog])
// 添加遵守该协议的类
delegate.add(cat1)
// 删除遵守该协议的类
delegate.remove(dog)

master.delegate = delegate
master.orderToEat()

// 输出
// Cat is eating meat
// Cat is eating meat
```

#### 2. 多重代理的应用场景
1. IM消息接收之后在多个地方做回调,比如显示消息,改变小红点,显示消息数.
2. `UISearchBar`的回调,当我们需要在多个地方获取数据的时候,类似的还有`UINavigationController`的回调等.


<h2 id="14">14.自动检查控制器是否被销毁</h2>  


检查内存泄漏除了使用`Instruments`,还有查看控制器`pop`或`dismiss`后是否被销毁,后者相对来说更方便一点.但老是盯着析构函数`deinit`看日志输出是否有点麻烦呢?


`UIViewController`有提供两个不知名的属性: 
 
 1. `isBeingDismissed`: 当modal出来的控制器被`dismiss`后的值为`true`.  
 2.  `isMovingFromParent`: 在控制器的堆栈中,如果当前控制器从父控制器中移除,值会变成`true`.


如果这两个属性都为`true`,表明控制器马上要被销毁了,但这是由ARC去做内存管理,我们并不知道多久之后被销毁,简单起见就设个2秒吧.

```swift
extension UIViewController {
    
    public func dch_checkDeallocation(afterDelay delay: TimeInterval = 2.0) {
        let rootParentViewController = dch_rootParentViewController
        
        if isMovingFromParent || rootParentViewController.isBeingDismissed {
            let disappearanceSource: String = isMovingFromParent ? "removed from its parent" : "dismissed"
            DispatchQueue.main.asyncAfter(deadline: .now() + delay, execute: { [weak self] in
                if let VC = self {
                    assert(self == nil, "\(VC.description) not deallocated after being \(disappearanceSource)")
                }
            })
        }
    }
    private var dch_rootParentViewController: UIViewController {
        var root = self
        while let parent = root.parent {
            root = parent
        }
        return root
    }
}
```
我们把这个方法添加到`viewDidDisappear(_:)`中

```swift
override func viewDidDisappear(_ animated: Bool) {
    super.viewDidDisappear(animated)
    
    dch_checkDeallocation()
}
```
如果发生循环引用,控制就不会被销毁,会触发`assert`报错.


<h2 id="15">15.向控制器中注入代码</h2>  

使用场景: 在某些控制器的`viewDidLoad`方法中,我们需要添加一段代码,用于统计某个页面的打开次数.  

最常用的解决方案:   
在父类或者`extension`中定义一个方法,然后在需要做统计的控制器的`viewDidLoad`方法中调用刚刚定义好的方法.

或者还可以使用代码注入.
#### 1. 代码注入的使用


```swift
ViewControllerInjector.inject(into: [ViewController.self], selector: #selector(UIViewController.viewDidLoad)) {

	// $0 为ViewController对象            
	// 统计代码...
}
```

#### 2.代码注入的实现

swift虽然是门静态语言,但依然支持OC的`runtime`.可以允许我们在静态类型中使用动态代码.代码注入就是通过`runtime`的交换方法实现的.

```swift
class ViewControllerInjector {
    
    typealias methodRef = @convention(c)(UIViewController, Selector) -> Void
    
    static func inject(into supportedClasses: [UIViewController.Type], selector: Selector, injection: @escaping (UIViewController) -> Void) {
        
        guard let originalMethod = class_getInstanceMethod(UIViewController.self, selector) else {
            fatalError("\(selector) must be implemented")
        }
        
        var originalIMP: IMP? = nil
        
        let swizzledViewDidLoadBlock: @convention(block) (UIViewController) -> Void = { receiver in
            if let originalIMP = originalIMP {
                let castedIMP = unsafeBitCast(originalIMP, to: methodRef.self)
                castedIMP(receiver, selector)
            }
            
            if ViewControllerInjector.canInject(to: receiver, supportedClasses: supportedClasses) {
                injection(receiver)
            }
        }
        
        let swizzledIMP = imp_implementationWithBlock(unsafeBitCast(swizzledViewDidLoadBlock, to: AnyObject.self))
        originalIMP = method_setImplementation(originalMethod, swizzledIMP)
    }
    
    
    private static func canInject(to receiver: Any, supportedClasses: [UIViewController.Type]) -> Bool {
        let supportedClassesIDs = supportedClasses.map { ObjectIdentifier($0) }
        let receiverType = type(of: receiver)
        return supportedClassesIDs.contains(ObjectIdentifier(receiverType))
    }
}
```
代码注入可以在不修改原有代码的基础上自定义自己所要的.相比继承,代码的可重用性会高一点,侵入性会小一点. 



<h2 id="16">16.给Extension添加存储属性</h2>  

我们都知道`Extension`中可以添加计算属性,但不能添加存储属性.  

对 我们可以使用`runtime`  

```swift
private var nameKey: Void?
extension UIView {
    // 给UIView添加一个name属性
    var name: String? {
        get {
            return objc_getAssociatedObject(self, &nameKey) as? String
        }
        set {
            objc_setAssociatedObject(self, &nameKey, newValue, .OBJC_ASSOCIATION_COPY_NONATOMIC)
        }
    }
}
```


<h2 id="17">17.用闭包实现按钮的链式点击事件</h2>  

#### 1. 通常姿势

通常按钮的点击事件我们需要这样写:

```swift
btn.addTarget(self, action: #selector(actionTouch), for: .touchUpInside)

@objc func actionTouch() {
    print("按钮点击事件")
}
```
如果有多个点击事件,往往还要写多个方法,写多了有没有觉得有点烦,代码阅读起来还要上下跳转.

#### 2. 使用闭包封装
##### 1. 实现  

```swift
private var actionDictKey: Void?
public typealias ButtonAction = (UIButton) -> ()

extension UIButton {
    
    // MARK: - 属性
    // 用于保存所有事件对应的闭包
    private var actionDict: (Dictionary<String, ButtonAction>)? {
        get {
            return objc_getAssociatedObject(self, &actionDictKey) as? Dictionary<String, ButtonAction>
        }
        set {
            objc_setAssociatedObject(self, &actionDictKey, newValue, .OBJC_ASSOCIATION_COPY_NONATOMIC)
        }
    }
    
    // MARK: - API
    @discardableResult
    public func addTouchUpInsideAction(_ action: @escaping ButtonAction) -> UIButton {
        self.addButton(action: action, for: .touchUpInside)
        return self
    }
    
    @discardableResult
    public func addTouchUpOutsideAction(_ action: @escaping ButtonAction) -> UIButton {
        self.addButton(action: action, for: .touchUpOutside)
        return self
    }
    
    @discardableResult
    public func addTouchDownAction(_ action: @escaping ButtonAction) -> UIButton {
        self.addButton(action: action, for: .touchDown)
        return self
    }
    
    // ...其余事件可以自己扩展
    
    // MARK: - 私有方法
    private func addButton(action: @escaping ButtonAction, for controlEvents: UIControl.Event) {
        
        let eventKey = String(controlEvents.rawValue)

        if var actionDict = self.actionDict {
            actionDict.updateValue(action, forKey: eventKey)
            self.actionDict = actionDict
        }else {
            self.actionDict = [eventKey: action]
        }

        switch controlEvents {
        case .touchUpInside:
            addTarget(self, action: #selector(touchUpInsideControlEvent), for: .touchUpInside)
        case .touchUpOutside:
            addTarget(self, action: #selector(touchUpOutsideControlEvent), for: .touchUpOutside)
        case .touchDown:
            addTarget(self, action: #selector(touchDownControlEvent), for: .touchDown)
        default:
            break
        }
    }
    
    // 响应事件
    @objc private func touchUpInsideControlEvent() {
        executeControlEvent(.touchUpInside)
    }
    @objc private func touchUpOutsideControlEvent() {
        executeControlEvent(.touchUpOutside)
    }
    @objc private func touchDownControlEvent() {
        executeControlEvent(.touchDown)
    }
    
    @objc private func executeControlEvent(_ event: UIControl.Event) {
        let eventKey = String(event.rawValue)
        if let actionDict = self.actionDict, let action = actionDict[eventKey] {
            action(self)
        }
    }
}

```

##### 2. 使用 

```swift
 btn
    .addTouchUpInsideAction { btn in
        print("addTouchUpInsideAction")
    }.addTouchUpOutsideAction { btn in
        print("addTouchUpOutsideAction")
    }.addTouchDownAction { btn in
        print("addTouchDownAction")
    }
```
##### 3. 实现原理
利用`runtime`在按钮的`extension`中添加一个字典属性,`key`对应的是事件类型,`value`对应的是该事件类型所要执行的闭包.然后再添加按钮的监听事件,在响应方法中,根据事件类型找到并执行对应的闭包.

链式调用就是不断返回自身.

有没有觉得如果这样做代码写起来会简洁一点呢?


<h2 id="18">18.用闭包实现手势的链式监听事件</h2>  

和tips17中的按钮点击事件类似,手势也可以封装成链式闭包回调.

#### 1. 使用

```swift
view
    .addTapGesture { tap in
        print(tap)
    }.addPinchGesture { pinch in
        print(pinch)
    }
```

#### 2. 实现过程

```swift
public typealias GestureClosures = (UIGestureRecognizer) -> Void
private var gestureDictKey: Void?

extension UIView {
    private enum GestureType: String {
        case tapGesture
        case pinchGesture
        case rotationGesture
        case swipeGesture
        case panGesture
        case longPressGesture
    }

    // MARK: - 属性
    private var gestureDict: [String: GestureClosures]? {
        get {
            return objc_getAssociatedObject(self, &gestureDictKey) as? [String: GestureClosures]
        }
        set {
            objc_setAssociatedObject(self, &gestureDictKey, newValue, .OBJC_ASSOCIATION_COPY_NONATOMIC)
        }
    }

    // MARK: - API
    /// 点击
    @discardableResult
    public func addTapGesture(_ gesture: @escaping GestureClosures) -> UIView {
        addGesture(gesture: gesture, for: .tapGesture)
        return self
    }
    /// 捏合
    @discardableResult
    public func addPinchGesture(_ gesture: @escaping GestureClosures) -> UIView {
        addGesture(gesture: gesture, for: .pinchGesture)
        return self
    }
   	 // ...省略相关手势
   	 
    // MARK: - 私有方法
    private func addGesture(gesture: @escaping GestureClosures, for gestureType: GestureType) {
        let gestureKey = String(gestureType.rawValue)
        if var gestureDict = self.gestureDict {
            gestureDict.updateValue(gesture, forKey: gestureKey)
            self.gestureDict = gestureDict
        } else {
            self.gestureDict = [gestureKey: gesture]
        }
        isUserInteractionEnabled = true
        switch gestureType {
        case .tapGesture:
            let tap = UITapGestureRecognizer(target: self, action: #selector(tapGestureAction(_:)))
            addGestureRecognizer(tap)
        case .pinchGesture:
            let pinch = UIPinchGestureRecognizer(target: self, action: #selector(pinchGestureAction(_:)))
            addGestureRecognizer(pinch)
        default:
            break
        }
    }
    @objc private func tapGestureAction (_ tap: UITapGestureRecognizer) {
        executeGestureAction(.tapGesture, gesture: tap)
    }
    @objc private func pinchGestureAction (_ pinch: UIPinchGestureRecognizer) {
        executeGestureAction(.pinchGesture, gesture: pinch)
    }
   
    private func executeGestureAction(_ gestureType: GestureType, gesture: UIGestureRecognizer) {
        let gestureKey = String(gestureType.rawValue)
        if let gestureDict = self.gestureDict, let gestureReg = gestureDict[gestureKey] {
            gestureReg(gesture)
        }
    }
}
```

具体实现 [猛击](https://github.com/DarielChen/SwiftTips/blob/master/SwiftTipsDemo/DCTool/Extension/UIView%2BExtension.swift)

<h2 id="19">19.用闭包实现通知的监听事件</h2>   

#### 1. 使用

```swift
// 通知监听
self.observerNotification(.notifyName1) { notify in
    print(notify.userInfo)
}
// 发出通知
self.postNotification(.notifyName1, userInfo: ["infoKey": "info"])

// 移除通知
self.removeNotification(.notifyName1)
```

#### 2. 实现

```swift
public typealias NotificationClosures = (Notification) -> Void
private var notificationActionKey: Void?

// 用于存放通知名称
public enum NotificationNameType: String {
    case notifyName1
    case notifyName2
}

extension NSObject {
    private var notificationClosuresDict: [NSNotification.Name: NotificationClosures]? {
        get {
            return objc_getAssociatedObject(self, &notificationActionKey)
                as? [NSNotification.Name: NotificationClosures]
        }
        set {
            objc_setAssociatedObject(self, &notificationActionKey, newValue, .OBJC_ASSOCIATION_COPY_NONATOMIC)
        }
    }
    public func postNotification(_ name: NotificationNameType, userInfo: [AnyHashable: Any]?) {
        NotificationCenter.default.post(name: NSNotification.Name(name.rawValue), object: self, userInfo: userInfo)
    }
    public func observerNotification(_ name: NotificationNameType, action: @escaping NotificationClosures) {
        if var dict = notificationClosuresDict {
            guard dict[NSNotification.Name(name.rawValue)] == nil else {
                return
            }
            dict.updateValue(action, forKey: NSNotification.Name(name.rawValue))
            self.notificationClosuresDict = dict
        } else {
            self.notificationClosuresDict = [NSNotification.Name(name.rawValue): action]
        }
        NotificationCenter.default.addObserver(self, selector: #selector(notificationAction),
                                               name: NSNotification.Name(name.rawValue), object: nil)
    }
    public func removeNotification(_ name: NotificationNameType) {
        NotificationCenter.default.removeObserver(self)
        notificationClosuresDict?.removeValue(forKey: NSNotification.Name(name.rawValue))
    }
    @objc func notificationAction(notify: Notification) {
        if let notificationClosures = notificationClosuresDict, let closures = notificationClosures[notify.name] {
            closures(notify)
        }
    }
}
```
具体实现过程和tips17、tips18类似.


<h2 id="20">20.AppDelegate解耦</h2>  

作为iOS整个项目的核心`App delegate`,随着项目的逐渐变大,会变得越来越臃肿,一不小心代码就过了千行.  

大型项目的`App delegate`体积会大到什么程度呢?我们可以参考下国外2亿多月活的`Telegram`的 [App delegate](https://github.com/peter-iakovlev/Telegram/blob/public/Telegraph/TGAppDelegate.mm).是不是吓一跳,4千多行.看到这样的代码是不是很想点击左上角的x.

是时候该给`App delegate`解耦了,目标: 每个功能的配置或者初始化都分开,各自做各自的事情.`App delegate`要做到只需要调用就好了.


#### 1.命令模式

> 命令模式: 发送方发送请求,然后接收方接受请求后执行,但发送方可能并不知道接受方是谁，执行的是什么操作,这样做的好处是发送方和接受方完全的松耦合，大大提高程序的灵活性.

##### 1. 定义好协议,把相关初始化配置代码分类

```swift
protocol Command {
    func execute()
}

struct InitializeThirdPartiesCommand: Command {
    func execute() {
        // 第三方库初始化代码
    }
}

struct InitialViewControllerCommand: Command {
    let keyWindow: UIWindow
    func execute() {
        // 根控制器设置代码
    }
}

struct InitializeAppearanceCommand: Command {
    func execute() {
        // 全局外观样式配置
    }
}

struct RegisterToRemoteNotificationsCommand: Command {
    func execute() {
        // 远程推送配置        
    }
}

```
##### 2. 管理者

```swift

final class StartupCommandsBuilder {
    private var window: UIWindow!
    
    func setKeyWindow(_ window: UIWindow) -> StartupCommandsBuilder {
        self.window = window
        return self
    }
    
    func build() -> [Command] {
        return [
            InitializeThirdPartiesCommand(),
            InitialViewControllerCommand(keyWindow: window),
            InitializeAppearanceCommand(),
            RegisterToRemoteNotificationsCommand()
        ]
    }
}
```

##### 3. `App delegate`调用

```swift
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        
        StartupCommandsBuilder()
            .setKeyWindow(window!)
            .build()
            .forEach { $0.execute() }
        
        return true
}
```

使用命令模式的好处是,如果要添加新的配置,设置完后只要加在`StartupCommandsBuilder`中就可以了.`App delegate`中不需要添加任何内容.

但这样做只能将`didFinishLaunchingWithOptions`中的代码解耦,`App delegate`中的其他方法怎样解耦呢?

#### 2.组合模式

> 组合模式: 可以将对象组合成树形结构来表现"整体/部分"层次结构. 组合后可以以一致的方法处理个别对象以及组合对象.

这边我们给`App delegate`每个功能模块都设置一个子类,每个子类包含所有`App delegate`的方法.

##### 1. 每个子模块实现各自的功能

```swift
// 推送
class PushNotificationsAppDelegate: AppDelegateType {
    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
        print("推送配置")
        return true
    }
    
    func application(_ application: UIApplication, didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
        print("推送相关代码...")
    }
    
    // 其余方法
}

// 外观样式
class AppearanceAppDelegate: AppDelegateType {
    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
        print("外观样式配置")
        return true
    }
}


// 控制器处理
class ViewControllerAppDelegate: AppDelegateType {
    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
        print("根控制器设置代码")
        return true
    }
}


// 第三方库
class ThirdPartiesConfiguratorAppDelegate: AppDelegateType {
    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
        print("第三方库初始化代码")
        return true
    }

    func applicationDidEnterBackground(_ application: UIApplication) {
        print("ThirdPartiesConfiguratorAppDelegate - applicationDidEnterBackground")
    }
    
    func applicationDidBecomeActive(_ application: UIApplication) {
        print("ThirdPartiesConfiguratorAppDelegate - applicationDidBecomeActive")
    }

}

```
##### 2. 管理者

```swift
typealias AppDelegateType = UIResponder & UIApplicationDelegate

class CompositeAppDelegate: AppDelegateType {
    private let appDelegates: [AppDelegateType]

    init(appDelegates: [AppDelegateType]) {
        self.appDelegates = appDelegates
    }

    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
        appDelegates.forEach { _ = $0.application?(application, didFinishLaunchingWithOptions: launchOptions) }
        return true
    }

    func application(_ application: UIApplication, didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
        appDelegates.forEach { _ = $0.application?(application, didRegisterForRemoteNotificationsWithDeviceToken: deviceToken) }
    }


    func applicationDidEnterBackground(_ application: UIApplication) {
        appDelegates.forEach { _ = $0.applicationDidEnterBackground?(application)
        }
    }

    func applicationDidBecomeActive(_ application: UIApplication) {
        appDelegates.forEach { _ = $0.applicationDidBecomeActive?(application)
        }
    }
}
```

##### 3. `App delegate`调用

```swift
@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {

    var window: UIWindow?
    let appDelegate = AppDelegateFactory.makeDefault()

    enum AppDelegateFactory {
        static func makeDefault() -> AppDelegateType {
            
            return CompositeAppDelegate(appDelegates: [
                PushNotificationsAppDelegate(),
                AppearanceAppDelegate(),
                ThirdPartiesConfiguratorAppDelegate(),
                ViewControllerAppDelegate(),
                ]
            )
        }
    }

    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
         _ = appDelegate.application?(application, didFinishLaunchingWithOptions: launchOptions)
        return true
    }

    func application(_ application: UIApplication, didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
        appDelegate.application?(application, didRegisterForRemoteNotificationsWithDeviceToken: deviceToken)
    }

    func applicationDidBecomeActive(_ application: UIApplication) {
        appDelegate.applicationDidBecomeActive?(application)
    }

    func applicationDidEnterBackground(_ application: UIApplication) {
        appDelegate.applicationDidEnterBackground?(application)
    }
}

```
`App delegate`解耦相比命令模式,使用组合模式可自定义程度会更高一点.

<h2 id="21">21.常见的编译器诊断指令</h2>  
swift标准库提供了很多编译器诊断指令,用于在编译阶段提前处理好相关事情.

下面列出了一些常见的编译器诊断指令:

##### 1. 代码中警告错误标识: #warning和#error
swift4.2中添加了这两个命令,再也不用在项目中自己配置错误和警告的命令了.  

警告⚠️:

```swift
// Xcode会报一条黄色警告
#warning("此处逻辑有问题,明天再说")

// TODO
#warning("TODO: Update this code for the new iOS 12 APIs")
```
错误❌:

```swift
// 手动设置一条错误
#error("This framework requires UIKit!")
```

##### 2. #if - #endif 条件判断

```swift
#if !canImport(UIKit)
#error("This framework requires UIKit!")
#endif

#if DEBUG
#warning("TODO: Update this code for the new iOS 12 APIs")
#endif        
```

##### 3. #file、#function、#line
分别用于获取文件名,函数名称,当前所在行数,一般用于辅助日志输出.

自定义Log

```swift
public struct dc {
    
    /// 自定义Log
    ///
    /// - Parameters:
    ///   - message: 输出的内容
    ///   - file: 默认
    ///   - method: 默认
    ///   - line: 默认
    public static func log<T>(_ message: T, file: NSString = #file, method: String = #function, line: Int = #line) {
        #if DEBUG
        print("\(file.pathComponents.last!):\(method)[\(line)]: \(message)")
        #endif
    }
}
```

##### 4. #available和@available

一般用来判断当前代码块在某个版本及该版本以上是否可用.

```swift
if #available(iOS 8, *) {
    // iOS 8 及其以上系统运行
}

guard #available(iOS 8, *) else {
    return //iOS 8 以下系统就直接返回
}

@available(iOS 11.4, *)
func myMethod() {
    // do something
}
```

<h2 id="22">22.最后执行的defer代码块</h2>  

`defer`这个关键字不是很常用,但有时还是很有用的.

具体用法,简而言之就是,`defer`代码块会在函数的return前执行.

```swift
func printStringNumbers() {
    defer { print("1") }
    defer { print("2") }
    defer { print("3") }
    
    print("4")
}
printStringNumbers() // 打印 4 3 2 1
```

下面列举几个常见的用途:

##### 1. try-catch

```swift
func foo() throws {
        
	defer {
	    print("two")
	}
	    
	do {
	    print("one")
	    throw NSError()
	    print("不会执行")
	}
	print("不会执行")
}


do {
    try foo()
} catch  {
    print("three")
}
// 打印 one two three
```
`defer`可在函数throw之后被执行,而如果将代码添加到`throw NSError()`底部和`do{}`底部都不会被执行.

##### 2. 文件操作

```swift
func writeFile() {
    let file: FileHandle? = FileHandle(forReadingAtPath: filepath)
    defer { file?.closeFile() }
    
    // 文件相关操作
}
```
这样一方面可读性好一点,另一方面不会因为某个地方throw了一个错误而没有关闭资源文件了.

##### 3. 避免忘记回调

```swift
func getData(completion: (_ result: Result<String>) -> Void) {
    var result: Result<String>?

    defer {
        guard let result = result else {
            fatalError("We should always end with a result")
        }
        completion(result)
    }

    // result的处理逻辑
}
```
`defer`中可以做一些result的验证逻辑,这样不会和result的处理逻辑混淆,代码清晰.


<h2 id="23">23.定义全局常量</h2>  

作为整个项目中通用的全局常量为了方便管理最好集中定义在一个地方.

下面介绍几种全局常量定义的姿势:

##### 1. 使用结构体

```swift
public struct Screen {
    static var width: CGFloat {
        return UIScreen.main.bounds.size.width
    }
    static var height: CGFloat {
        return UIScreen.main.bounds.size.height
    }
    static var statusBarHeight: CGFloat {
        return UIApplication.shared.statusBarFrame.height
    }
}

Screen.width  // 屏幕宽度
Screen.height // 屏幕高度
Screen.statusBarHeight // statusBar高度
```

好处是能比较直观的看出全局常量的定义逻辑,方便后面扩展.

##### 2. 使用没有case的枚举

正常情况下的`enum`都是与`case`搭配使用,如果使用了`case`就要实例化`enum`.其实也可以不写`case`.

```swift
public enum ConstantsEnum {
     static let width: CGFloat = 100
     static let height: CGFloat = 50
}

ConstantsEnum.width

let instance = ConstantsEnum()
// ERROR: 'ConstantsEnum' cannot be constructed because it has no accessible initializers
```
`ConstantsEnum`不可以实例化,会报错.

相比`struct`,使用枚举定义常量可以避免不经意间实例化对象.

##### 3. 使用extension
使用`extension`几乎可以为任何类型扩展常量.

例如,通知名称

```swift
public extension Notification.Name {
    // 名称
    static let customNotification = Notification.Name("customNotification")
}

NotificationCenter.default.post(name: .customNotification, object: nil)
```

增加自定义颜色

```swift
public extension UIColor {
    class var myGolden: UIColor {
        return UIColor(red: 1.000, green: 0.894, blue: 0.541, alpha: 0.900)
    }
}

view.backgroundColor = .myGolden
```

增加double常量

```swift
public extension Double {
    public static let kRectX = 30.0
    public static let kRectY = 30.0
    public static let kRectWidth = 30.0
    public static let kRectHeight = 30.0
}

CGRect(x: .kRectX, y: .kRectY, width: .kRectWidth, height: .kRectHeight)
```

因为传入参数类型是确定的,我们可以把类型名省略,直接用点语法.


<h2 id="24">24.使用Codable协议解析JSON</h2>  

swift4.0推出的`Codable`协议用来解析JSON还是挺不错的.

##### 1. JSON、模型互相转化

```swift
public protocol Decodable { 
	public init(from decoder: Decoder) throws 
}
public protocol Encodable { 
	public func encode(to encoder: Encoder) throws 
} 
public typealias Codable = Decodable & Encodable
```
`Codable`是`Decodable`和`Encodable`这两个协议的综合,只要遵守了`Codable `协议,编译器就能帮我们实现好一些细节,然后就可以做编码和解码操作了.

```swift
public struct Pet: Codable {
    var name: String
    var age: Int
}

let json = """
    [{
        "name": "WangCai",
        "age": 2,
    },{
        "name": "xiaoHei",
        "age": 3,
    }]
    """.data(using: .utf8)!
    
// JSON -> 模型
let decoder = JSONDecoder()
do {
    // 对于数组可以使用[Pet].self
    let dogs = try decoder.decode([Pet].self, from: json)
    print(dogs)
}catch {
    print(error)
}
    
// 模型 -> JSON
let encoder = JSONEncoder()
encoder.outputFormatting = .prettyPrinted // 美化样式
do {
    let data = try encoder.encode(Pet(name: "XiaoHei", age: 3))
    print(String(data: data, encoding: .utf8)!)
//      {
//          "name" : "XiaoHei",
//          "age" : 3
//      }
} catch {
    print(error)
}
```

##### 2. `Codable`做了哪些事情

下面我们重写系统的方法.

```swift
init(name: String, age: Int) {  
    self.name = name  
    self.age = age
}

// decoding 
init(from decoder: Decoder) throws { 
    let container = try decoder.container(keyedBy: CodingKeys.self)  
    let name = try container.decode(String.self, forKey: .name)  
    let age = try container.decode(Int.self, forKey: .age)  
    self.init(name: name, age: age) 
} 

// encoding 
func encode(to encoder: Encoder) throws {  
    var container = encoder.container(keyedBy: CodingKeys.self)  
    try container.encode(name, forKey: .name)  
    try container.encode(age, forKey: .age) 
}

enum CodingKeys: String, CodingKey {  
    case name  
    case age
}
```

对于编码和解码的过程,我们都是创建一个容器,该容器有一个`keyedBy`的参数,用于指定属性和JSON中key两者间的映射的规则,因此我们传`CodingKeys`的类型过去,说明我们要使用该规则来映射.对于解码的过程,我们使用该容器来进行解码,指定要值的类型和获取哪一个key的值,同样的,编码的过程中,我们使用该容器来指定要编码的值和该值对应json中的key.

##### 3. `Codable`实际使用场景

当然了,现实开发中需要解析的JSON不会这么简单.

```swift

let json = """
        {
            "aircraft": {
                "identification": "NA875",
                "color": "Blue/White"
            },
            "route": ["KTTD", "KHIO"],
            "departure_time": {
                "proposed": 1540868946509,
                "actual": 1540869946509,
            },
            "flight_rules": "IFR",
            "remarks": null,
            "price": "NaN",
        }
        """.data(using: .utf8)!


public struct Aircraft: Codable {
    public var identification: String
    public var color: String
}
public enum FlightRules: String, Codable {
    case visual = "VFR"
    case instrument = "IFR"
}
public struct FlightPlan: Codable {
    // 嵌套模型
    public var aircraft: Aircraft
    // 包含数组
    public var route: [String]
    // 日期处理
    private var departureTime: [String: Date]
    public var proposedDepartureDate: Date? {
        return departureTime["proposed"]
    }
    public var actualDepartureDate: Date? {
        return departureTime["actual"]
    }
    // 枚举处理
    public var flightRules: FlightRules
    // 空值处理
    public var remarks: String?
    // 特殊值处理
    public var price: Float
    // 下划线key转驼峰命名
    private enum CodingKeys: String, CodingKey {
        case aircraft
        case flightRules = "flight_rules"
        case route
        case departureTime = "departure_time"
        case remarks
        case price
    }
}


let decoder = JSONDecoder()

// 解码时,日期格式是13位时间戳 .base64:通过base64解码
decoder.dateDecodingStrategy = .millisecondsSince1970

// 指定 infinity、-infinity、nan 三个特殊值的表示方式
decoder.nonConformingFloatDecodingStrategy = .convertFromString(positiveInfinity: "+∞", negativeInfinity: "-∞", nan: "NaN")

do {
    let plan = try decoder.decode(FlightPlan.self, from: json)
    
    plan.aircraft.color // Blue/White
    plan.aircraft.identification // NA875
    plan.route // ["KTTD", "KHIO"]
    plan.proposedDepartureDate // 2018-10-30 03:09:06 +0000
    plan.actualDepartureDate // 2018-10-30 03:25:46 +0000
    plan.flightRules // instrument
    plan.remarks // 可选类型 空
    plan.price // nan
    
}catch {
    print(error)
}

```

swift4.1中有个属性可以自动将key转化为驼峰命名:
`decoder.keyDecodingStrategy = .convertFromSnakeCase`,如果`CodingKeys`只是用来转成驼峰命名的话,设置好这个属性后就可以不用写`CodingKeys`这个枚举了.


<h2 id="25">25.dispatch_once替代方案</h2>  

OC中用来保证代码块只执行一次的`dispatch_once`在swfit中已经被废弃了,取而代之的是使用`static let`,`let`本身就带有线程安全性质的.

例如单例的实现.

```swift
final public class MySingleton {
    static let shared = MySingleton()
    private init() {}
}
```
但如果我们不想定义常量,需要某个代码块执行一次呢?

```swift
private lazy var takeOnceTime: Void = {
    // 代码块...
}()

_ = takeOnceTime
```
定义一个懒加载的变量,防止在初始化的时候被执行.后面加一个`void`,为了在`_ = takeOnceTime`赋值时不耗性能,返回一个`Void`类型.  

`lazy var`改为`static let`也可以,为了使用方便,我们用一个类方法封装下

```swift
class ClassName {
    private static let takeOnceTime: Void = {
        // 代码块...
    }()
    static func takeOnceTimeFunc() {
        ClassName.takeOnceTime
    }
}

// 使用
ClassName.takeOnceTimeFunc()
```
这样就可以做到和`dispatch_once`一样的效果了.


<h2 id="26">26.被废弃的+load()和+initialize()</h2>  

我们都知道OC中两个方法`+load()`和`+initialize()`. 
 
`+load()`:  app启动的时候会加载所有的类,此时就会调用每个类的load方法.  
`+initialize()`: 第一次初始化这个类的时候会被调用.  

然而在目前的swift版本中这两个方法都不可用了,那现在我们要在这个阶段搞事情该怎么做? 例如`method swizzling`.

[JORDAN SMITH大神](http://jordansmith.io/handling-the-deprecation-of-initialize/)给出了一种很巧解决方案.`UIApplication`有一个`next`属性,它会在`applicationDidFinishLaunching`之前被调用,这个时候通过`runtime`获取到所有类的列表,然后向所有遵循SelfAware协议的类发送消息.

```swift
extension UIApplication {
    private static let runOnce: Void = {
        NothingToSeeHere.harmlessFunction()
    }()
    override open var next: UIResponder? {
        // Called before applicationDidFinishLaunching
        UIApplication.runOnce
        return super.next
    }
}

protocol SelfAware: class {
    static func awake()
}
class NothingToSeeHere {
    static func harmlessFunction() {
        let typeCount = Int(objc_getClassList(nil, 0))
        let types = UnsafeMutablePointer<AnyClass>.allocate(capacity: typeCount)
        let autoreleasingTypes = AutoreleasingUnsafeMutablePointer<AnyClass>(types)
        objc_getClassList(autoreleasingTypes, Int32(typeCount))
        for index in 0 ..< typeCount {
            (types[index] as? SelfAware.Type)?.awake()
        }
        types.deallocate()
    }
}
```

之后任何遵守`SelfAware`协议实现的`+awake()`方法在这个阶段都会被调用.


 <h2 id="27">27.交换方法 Method Swizzling</h2>  

黑魔法`Method Swizzling`在swift中实现的两个困难点

- swizzling 应该保证只会执行一次.
- swizzling 应该在加载所有类的时候调用.

分别在`tips25`和`tips26`中给出了解决方案.

下面给出了两个示例供参考:

```swift
protocol SelfAware: class {
    static func awake()
    static func swizzlingForClass(_ forClass: AnyClass, originalSelector: Selector, swizzledSelector: Selector)
}

extension SelfAware {
    
    static func swizzlingForClass(_ forClass: AnyClass, originalSelector: Selector, swizzledSelector: Selector) {
        let originalMethod = class_getInstanceMethod(forClass, originalSelector)
        let swizzledMethod = class_getInstanceMethod(forClass, swizzledSelector)
        guard (originalMethod != nil && swizzledMethod != nil) else {
            return
        }
        if class_addMethod(forClass, originalSelector, method_getImplementation(swizzledMethod!), method_getTypeEncoding(swizzledMethod!)) {
            class_replaceMethod(forClass, swizzledSelector, method_getImplementation(originalMethod!), method_getTypeEncoding(originalMethod!))
        } else {
            method_exchangeImplementations(originalMethod!, swizzledMethod!)
        }
    }
}

class NothingToSeeHere {
    static func harmlessFunction() {
        let typeCount = Int(objc_getClassList(nil, 0))
        let types = UnsafeMutablePointer<AnyClass>.allocate(capacity: typeCount)
        let autoreleasingTypes = AutoreleasingUnsafeMutablePointer<AnyClass>(types)
        objc_getClassList(autoreleasingTypes, Int32(typeCount))
        for index in 0 ..< typeCount {
            (types[index] as? SelfAware.Type)?.awake()
        }
        types.deallocate()
    }
}
extension UIApplication {
    private static let runOnce: Void = {
        NothingToSeeHere.harmlessFunction()
    }()
    override open var next: UIResponder? {
        UIApplication.runOnce
        return super.next
    }
}
```
在`SelfAware`的`extension`中为`swizzlingForClass`做了默认实现,相当于一层封装.

###### 1. 给按钮添加点击计数

```swift
extension UIButton: SelfAware {
    static func awake() {
        UIButton.takeOnceTime
    }
    private static let takeOnceTime: Void = {
        let originalSelector = #selector(sendAction)
        let swizzledSelector = #selector(xxx_sendAction(action:to:forEvent:))
        
        swizzlingForClass(UIButton.self, originalSelector: originalSelector, swizzledSelector: swizzledSelector)
    }()
    
    @objc public func xxx_sendAction(action: Selector, to: AnyObject!, forEvent: UIEvent!) {
        struct xxx_buttonTapCounter {
            static var count: Int = 0
        }
        xxx_buttonTapCounter.count += 1
        print(xxx_buttonTapCounter.count)
        xxx_sendAction(action: action, to: to, forEvent: forEvent)
    }
}
```
###### 2. 替换控制器的`viewWillAppear`方法

```swift
extension UIViewController: SelfAware {
    static func awake() {
        swizzleMethod
    }
    private static let swizzleMethod: Void = {
        let originalSelector = #selector(viewWillAppear(_:))
        let swizzledSelector = #selector(swizzled_viewWillAppear(_:))
        
        swizzlingForClass(UIViewController.self, originalSelector: originalSelector, swizzledSelector: swizzledSelector)
    }()
    
    @objc func swizzled_viewWillAppear(_ animated: Bool) {
        swizzled_viewWillAppear(animated)
        print("swizzled_viewWillAppear")
    }
}
```
<h2 id="28">28.获取View的指定子视图</h2>  

通过递归获取指定`view`的所有子视图.

#### 1. 获取`View`的子视图

使用

```swift
    let subViewArr = view.getAllSubViews() // 获取所有子视图
    let imageViewArr = view.getSubView(name: "UIImageView") // 获取指定类名的子视图
```

实现

```swift
extension UIView {
    private static var getAllsubviews: [UIView] = []
    public func getSubView(name: String) -> [UIView] {
        let viewArr = viewArray(root: self)
        UIView.getAllsubviews = []
        return viewArr.filter {$0.className == name}
    }
    public func getAllSubViews() -> [UIView] {
        UIView.getAllsubviews = []
        return viewArray(root: self)
    }
    private func viewArray(root: UIView) -> [UIView] {
        for view in root.subviews {
            if view.isKind(of: UIView.self) {
                UIView.getAllsubviews.append(view)
            }
            _ = viewArray(root: view)
        }
        return UIView.getAllsubviews
    }
}

extension NSObject {
    var className: String {
        let name = type(of: self).description()
        if name.contains(".") {
            return name.components(separatedBy: ".")[1]
        } else {
            return name
        }
    }
}

```

#### 2. 获取UIAlertController的titleLabel和messageLabel
`UIAlertController`好用,但可自定义程度不高,例如我们想让`message`文字左对齐,就需要获取到`messageLabel`,但`UIAlertController`并没有提供这个属性.

我们就可以通过递归拿到`alertTitleLabel`和`alertMessageLabel`.

```swift
extension UIAlertController {
    public var alertTitleLabel: UILabel? {
        return self.view.getSubView(name: "UILabel").first as? UILabel
    }
    public var alertMessageLabel: UILabel? {
        return self.view.getSubView(name: "UILabel").last as? UILabel
    }
}
```
虽然通过这种方法可以拿到`alertTitleLabel`和`alertMessageLabel`.但没法区分哪个是哪个,`alertTitleLabel`为默认子控件的第一个`label`,如果`title`传空,`message`传值,`alertTitleLabel`和`alertMessageLabel`获取到的都是`message`的`label`.

如果有更好的方法欢迎讨论.


<h2 id="29">29.线程安全: 互斥锁和自旋锁(10种)</h2>  

无并发,不编程.提到多线程就很难绕开锁🔐.

iOS开发中较常见的两类锁:  
###### 1. 互斥锁: 同一时刻只能有一个线程获得互斥锁,其余线程处于挂起状态.
###### 2. 自旋锁: 当某个线程获得自旋锁后,别的线程会一直做循环,尝试加锁,当超过了限定的次数仍然没有成功获得锁时,线程也会被挂起.

自旋锁较适用于锁的持有者保存时间较短的情况下,实际使用中互斥锁会用的多一些.

#### 1. 互斥锁,信号量

##### 1.遵守`NSLocking`协议的四种锁

四种锁分别是:  
`NSLock`、`NSConditionLock`、`NSRecursiveLock`、`NSCondition`
 
`NSLocking`协议
 
```swift
public protocol NSLocking {    
    public func lock()
    public func unlock()
}
```

下面举个多个售票点同时卖票的例子

```swift
var ticket = 20
var lock = NSLock()
    
override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
    let thread1 = Thread(target: self, selector: #selector(saleTickets), object: nil)
    thread1.name = "售票点A"
    thread1.start()
    
    let thread2 = Thread(target: self, selector: #selector(saleTickets), object: nil)
    thread2.name = "售票点B"
    thread2.start()
}
	
@objc private func saleTickets() {
	while true {
	    lock.lock()
	    Thread.sleep(forTimeInterval: 0.5) // 模拟延迟
	    if ticket > 0 {
	        ticket = ticket - 1
	        print("\(String(describing: Thread.current.name!)) 卖出了一张票,当前还剩\(ticket)张票")
	        lock.unlock()
	    }else {
	        print("oh 票已经卖完了")
	        lock.unlock()
	        break;
	    }
	}
}
```
遵守协议后实现的两个方法`lock()`和`unlock()`,意如其名.  

除此之外`NSLock`、`NSConditionLock`、`NSRecursiveLock`、`NSCondition`四种互斥锁各有其实现: 
###### 1. 除`NSCondition`外,三种锁都有的两个方法:

```swift
    // 尝试去锁,如果成功,返回true,否则返回false
    open func `try`() -> Bool
    // 在limit时间之前获得锁,没有返回NO
    open func lock(before limit: Date) -> Bool
```
###### 2. `NSCondition`条件锁:

```swift
    // 当前线程挂起
    open func wait()
    // 当前线程挂起,设置一个唤醒时间
    open func wait(until limit: Date) -> Bool
    // 唤醒在等待的线程
    open func signal()
    // 唤醒所有NSCondition挂起的线程
    open func broadcast()
```

当调用`wait()`之后,`NSCondition`实例会解锁已有锁的当前线程,然后再使线程休眠,当被`signal()`通知后,线程被唤醒,然后再给当前线程加锁,所以看起来好像`wait()`一直持有该锁,但根据苹果文档中说明,直接把`wait()`当线程锁并不能保证线程安全.

###### 3. `NSConditionLock `条件锁:

`NSConditionLock`是借助`NSCondition`来实现的,在`NSCondition`的基础上加了限定条件,可自定义程度相对`NSCondition`会高些.

```swift
    // 锁的时候还需要满足condition
    open func lock(whenCondition condition: Int)
    // 同try,同样需要满足condition
    open func tryLock(whenCondition condition: Int) -> Bool
    // 同unlock,需要满足condition
    open func unlock(withCondition condition: Int)
    // 同lock,需要满足condition和在limit时间之前
    open func lock(whenCondition condition: Int, before limit: Date) -> Bool
```

###### 4. `NSRecurisiveLock`递归锁:
定义了可以多次给相同线程上锁并不会造成死锁的锁.

提供的几个方法和`NSLock`类似.



##### 2. GCD的`DispatchSemaphore`和栅栏函数

###### 1. `DispatchSemaphore`信号量:

`DispatchSemaphore`中的信号量,可以解决资源抢占的问题,支持信号的通知和等待.每当发送一个信号通知,则信号量+1;每当发送一个等待信号时信号量-1,如果信号量为0则信号会处于等待状态.直到信号量大于0开始执行.所以我们一般将`DispatchSemaphore`的value设置为1.

下面给出了`DispatchSemaphore`的封装类

```swift
class GCDSemaphore {
    // MARK: 变量
    fileprivate var dispatchSemaphore: DispatchSemaphore!
    // MARK: 初始化
    public init() {
        dispatchSemaphore = DispatchSemaphore(value: 0)
    }
    public init(withValue: Int) {
        dispatchSemaphore = DispatchSemaphore(value: withValue)
    }
    // 执行
    public func signal() -> Bool {
        return dispatchSemaphore.signal() != 0
    }
    public func wait() {
        _ = dispatchSemaphore.wait(timeout: DispatchTime.distantFuture)
    }
    public func wait(timeoutNanoseconds: DispatchTimeInterval) -> Bool {
        if dispatchSemaphore.wait(timeout: DispatchTime.now() + timeoutNanoseconds) == DispatchTimeoutResult.success {
            return true
        } else {
            return false
        }
    }
}
```
###### 2. `barrier`栅栏函数:
栅栏函数也可以做线程同步,当然了这个肯定是要并行队列中才能起作用.只有当当前的并行队列执行完毕,才会执行栅栏队列.

```swift
/// 创建并发队列
let queue = DispatchQueue(label: "queuename", attributes: .concurrent)
/// 异步函数
queue.async {
    for _ in 1...5 {
        print(Thread.current)
    }
}
queue.async {
    for _ in 1...5 {
        print(Thread.current)
    }
}
/// 栅栏函数
queue.async(flags: .barrier) {
    print("barrier")
}
queue.async {
    for _ in 1...5 {
        print(Thread.current)
    }
}
```

##### 3. 其他的互斥锁
###### 1. `pthread_mutex`互斥锁
`pthread`表示`POSIX thread`,跨平台的线程相关的API,`pthread_mutex`也是一种互斥锁,互斥锁的实现原理与信号量非常相似,阻塞线程并睡眠,需要进行上下文切换.

一般情况下,一个线程只能申请一次锁,也只能在获得锁的情况下才能释放锁,多次申请锁或释放未获得的锁都会导致崩溃.假设在已经获得锁的情况下再次申请锁,线程会因为等待锁的释放而进入睡眠状态,因此就不可能再释放锁，从而导致死锁.

这边给出了一个基于`pthread_mutex_t`(安全的"FIFO"互斥锁)的封装 [MutexLock](https://github.com/DarielChen/SwiftTips/blob/master/SwiftTipsDemo/DCTool/DCTool/MutexLock.swift)


###### 1. @synchronized条件锁
日常开发中最常用的应该是@synchronized,这个关键字可以用来修饰一个变量,并为其自动加上和解除互斥锁.这样,可以保证变量在作用范围内不会被其他线程改变.但是在swift中它已经不存在了.其实@synchronized在幕后做的事情是调用了`objc_sync`中的`objc_sync_enter`和`objc_sync_exit` 方法，并且加入了一些异常判断.

因此我们可以利用闭包自己封装一套.

```swift
func synchronized(lock: AnyObject, closure: () -> ()) {
    objc_sync_enter(lock)
    closure()
    objc_sync_exit(lock)
}

// 使用
synchronized(lock: AnyObject) {
    // 此处AnyObject不会被其他线程改变
}

```

#### 2. 自旋锁

###### 1. `OSSpinLock`自旋锁
`OSSpinLock`是执行效率最高的锁,不过在iOS10.0以后已经被废弃了.

详见大神ibireme的[不再安全的 OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)

###### 2. `os_unfair_lock`自旋锁

它能够保证不同优先级的线程申请锁的时候不会发生优先级反转问题.这是苹果为了取代`OSSPinLock`新出的一个能够避免优先级带来的死锁问题的一个锁,`OSSPinLock`就是有由于优先级造成死锁的问题.

注意: 这个锁适用于小场景下的一个高效锁,否则会大量消耗cpu资源.

```swift
var unsafeMutex = os_unfair_lock()
os_unfair_lock_lock(&unsafeMutex)
os_unfair_lock_trylock(&unsafeMutex)
os_unfair_lock_unlock(&unsafeMutex)
```

这边给出了基于`os_unfair_lock`的封装 [MutexLock](https://github.com/DarielChen/SwiftTips/blob/master/SwiftTipsDemo/DCTool/DCTool/MutexLock.swift)

#### 3. 性能比较

这边贴一张大神ibireme在iPhone6、iOS9对各种锁的性能测试图

![](http://pcb5zz9k5.bkt.clouddn.com/lock_benchmark.png)

参考:  
[不再安全的OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)  
[深入理解iOS开发中的锁](https://bestswifter.com/ios-lock/)