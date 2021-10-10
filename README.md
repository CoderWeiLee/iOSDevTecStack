# iOSDevTecStack
iOS开发者需要具备的技术栈各种知识点收集汇总整理，不断完善中，欢迎提issue/PR

## Swift
#### dynamic
- 被`@objc dynamic` 修饰的内容会具有动态性，比如调用方法会走`runtime`那一套流程
```objc
  class Dog: NSObject {
    @objc dynamic func test1() {}
    func test2() {}
  }
  var d = Dog()
  d.test1()
  d.test2()
```

```objc
  movq 0x8fb4(%rip), %rsi       ; “test1”
  movq -0x60(%rip), %rax
  movq %rax, %rdi
  callq  0x100007c5e            ; symbol stub for: objc_msgSend
```

```objc
  movq -0x70(%rbp), %rcx
  movq (%rcx), %rdx
  andq (%rax), %rdx
  movq %rcx, %r13
  callq *0x50(%rdx)   //test2通过虚表方式调用
```

#### Swift支持KVC\KVO的条件
- 属性所在的类、监听器最终继承自`NSObject`
- 用`@objc dynamic`修饰对应的属性

#### static
- 修饰的属性变量是静态的，只会初始化一次
```objc
  class ViewController: UIViewController {
    static var age: Int = {return 0}
  }
```
> 类型属性`age`本质上就是全局变量，只会初始化一次，默认是`lazy`的。当`ViewController`被销毁了，再次进来`ViewController`重新创建的时候，`age`也不会再次创建，因为它是全局变量。所以可以用来实现类似`dispatch_onece`的效果，即整个程序运行过程中只执行一次的代码

#### @discardableResult
```objc
  @discardableResult
  func test() -> DispatchWorkItem {
    return DispatchWorkItem {
        print(1)
    }
  }
  test()
```
> 可丢弃的结果

#### 实现`dispatch_onece`的方式
```objc
  fileprivate var initTask: Void = {
   print("init--------")
}()

class aViewController: UIViewController {
//    static var initTask: Void = {
//       print("getAge")
//    }()
    
    override func viewDidLoad() {
        super.viewDidLoad()
//        let _ = Self.initTask
//        let _ = Self.initTask
        let _ = initTask
        let _ = initTask
    }
}
```
> `bt`查看调用栈会发现底层都调用了`dispatch_onece`方法

#### Swift加锁
```objc
  public struct Cache {
    private static var data = [String: Any]()
//    private static var lock = DispatchSemaphore(value: 1) // 1代表同时只能有1个线程访问
    private static var lock = NSLock()
//    private static var lock = NSRecursiveLock() //递归锁
    public static func get(_ key: String) -> Any? {
        data[key]
    }
    
    public static func set(_ key: String, _ value: Any) {
//        lock.wait()
//        defer {lock.signal()}
//        data[key] = value
        lock.lock()
        defer {lock.unlock()} //递归可能引发死锁
        data[key] = value
        
    }
}
```
#### 查看`View`的递归子控件
```objc
  (lldb) po [self.view.window recursiveDescription]
<UIWindow: 0x12c709ec0; frame = (0 0; 390 844); gestureRecognizers = <NSArray: 0x600003d2c1e0>; layer = <UIWindowLayer: 0x600003d22910>>
   | <UITransitionView: 0x12c506760; frame = (0 0; 390 844); autoresize = W+H; layer = <CALayer: 0x600003373960>>
   |    | <UIDropShadowView: 0x12c704e50; frame = (0 0; 390 844); autoresize = W+H; layer = <CALayer: 0x600003378b00>>
   |    |    | <UIView: 0x12c506e60; frame = (0 0; 390 844); autoresize = W+H; layer = <CALayer: 0x600003373440>>
```

#### Array的常用操作
- reduce
```objc
  var arr = [1,2,3]
  arr.reduce(0) {$0 + $1}
  arr.reduce(0, +) 
```

- flatMap
```objc
  arr.flatMap {
    return Array(repeating: $0, count: $0) //[1,2,2,3,3,3]
  }
```

- compactMap
```objc
  ["jack", "2", "3", "nick"].compactMap {Int($0)} // [2,3]
```
> 如果采用`map {Int($0)}` 返回的结果是`[nil,Optional(2),Optional(3),nil]`

- 采用reduce来实现map、filter的功能
```objc
  var arr = [1,2,3]
  print(arr.map {$0 * 2})
  print(arr.reduce([], {$0 + [$1 * 2]}))

  print(arr.filter {$0 % 2 == 0})
  print(arr.reduce([]) {$1 % 2 == 0 ? $0 + [$1] : $0})
```

#### lazy的map
- 普通的map
```objc
  let arr = [1,2,3]
  let result = arr.map {
    (i: Int) -> Int in
    print(“mapping \(i)”)
    return i * 2
  }
  print(“begin----”)
  print(“mapped”,result[0])
  print(“mapped”,result[1])
  print(“mapped”,result[2])
  print(“end----”)
```
>  普通的`map`会在begin开始之前完成所有映射，造成性能浪费

- lazy的map
```objc
  let arr = [1,2,3]
  let result = arr.lazy.map { //arr是sequence  arr.lazy是lazySequence
    (i: Int) -> Int in
    print(“mapping \(i)”)
    return i * 2
  }
  print(“begin----”)
  print(“mapped”,result[0])
  print(“mapped”,result[1])
  print(“mapped”,result[2])
  print(“end----”)
```
> 使用了`lazy`只有在用到一个值时才会去映射

#### Optional的map和flatMap
```objc
  var num1: Int? = 10
  //Optional(20)
  var num2 = num1.map { $0 * 2 }

  var num3: Int? = nil
  //nil
  var num4 = num3.map { $0 * 2 }
```

```objc
  var num1: Int? = 10
  //Optional(Optional(10))
  var num2 = num1.map { Optional.some($0 * 2)}
  //Optional(20)
  var num3 = num1.flatMap { Optional.some($0 * 2)}
```

```objc
  var num1: Int? = 10
  var num2 = (num1 != nil) ? (num1! + 10) : nil
  var num3 = num1.map { $0 + 10}
//num2,num3是等价的
```

```objc
  var fmt = DateFormatter()
  fmt.dateFormat = "yyyy-MM-dd"
  var str: String? = "2021-09-15"
  //old
  var date1 = str != nil ? fmt.date(from: str!) : nil
  //new
  var date2 = str.flatMap(fmt.date)
```

#### 实现只能被类遵守的协议
```objc
  protocol Runnable: Anyobject {}
  protocol Runnable: class {}
  @objc protocol Runnable {}
```
> @objc定义的协议，也可以被OC的类去遵守实现

#### 实现可选协议的方式
- 通过`Extension`提供协议中方法的默认实现
- 通过以下代码
```objc
   @objc protocol Runnable {
     @objc optional func run1
     func run2
   }
```
> 其中，定义的run1就是可选的方法

#### 函数的合成
```objc
  //函数合成
  prefix func ~<A, B, C, D>(_ fn: @escaping (A, B, C) -> D) -> (C) -> (B) -> (A) -> D  {
    {c in {b in {a in fn(a, b, c)}}}
  }

```

#### 优雅的前缀
```objc
  import Foundation
//前缀类型
struct MJ<Base> {
    var base: Base
    init(_ base: Base) {
        self.base = base
    }
}

//利用协议扩展前缀属性
protocol MJCompatible {}
extension MJCompatible {
    var mj: MJ<Self> {
        set {}
        get { MJ(self) }
    }
    static var mj: MJ<Self>.Type {
        set {}
        get { MJ<Self>.self }
    }
}
//给字符串拓展功能
//让String拥有mj前缀属性
extension String: MJCompatible {}

class Person {}

extension Person: MJCompatible {}
//给String.mj  String().mj前缀拓展功能
extension MJ where Base: ExpressibleByStringLiteral {
    var numberCount: Int {
        var count = 0
        let string = base as! String
        for c in string where ("0"..."9").contains(c) {
            count += 1
        }
        return count
    }
    
    static func test() {
        
    }
    mutating func run() {
        
    }
}

extension MJ where Base: Person {
    func run() {
        
    }
}

var person = Person()
person.mj.run()
print("123ccc".mj.numberCount)
String.mj.test()
var abc = "123"
abc.mj.run()

```

## Shell
#### ssh登录远程服务器
```objc
ssh -p 188 root@171.17.12.80 //step 1
/*Enter Password */          //step 2
```
> -p 188指定端口号

#### Mac终端命令远程开启屏幕共享进行远程控制
> 远程被控制的那台机器需要开启允许远程登录。系统偏好设置 -> 共享 -> 远程登录（勾选上）-> 允许远程用户对磁盘进行完全访问。开启后，你会看到`若要远程登录这台电脑，请键入“ssh 你的用户名@192.168.xx.xx”。 `
- 在自己的机器上，`ssh`登录远程主机。输入上面的命令回车然后再输入你远程主机的开机密码。
```objc
  ssh liwei@192.168.xx.xx
```
- 远程登录成功后。会看到远程主机的名字出现。
```objc
  liwei@liweideiMac
```
- 执行开启命令。执行的命令其实就是修改一个系统屏幕分享的配置文件。具体方式如下：
```objc
  sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart -activate -configure -access -on -clientopts -setvnclegacy -vnclegacy yes -clientopts -setvncpw -vncpw 1234 -restart -agent -privs -all
```
- 设置完之后会看到以下输出:
```objc
  Starting...
  Warning: macos 10.14 and later only allows control if Screen            Sharing is enabled through System Preferences.
   Activated Remote Management.
  Stopped ARD Agent.
  liwei: Set user remote control privileges.
  liwei: Set user remote access.
  Set the client options.
  Done.
```
- 开始远程控制。Mac 自带支持VNC，可以直接用系统浏览器Safari也可以使用支持VNC的第三方软件来进行控制。使用Safari控制的方式为：
     - 打开Safari
     - 在地址栏里输入vnc+远程主机地址
     ```objc
       vnc://10.5.234.xx
     ```
     - 回车后输入远程地址的用户名和密码
     - 回车进行连接
   > 到这一步应该已经可以看到远程主机的屏幕了。下面的是为所有用户开启vnc和关闭的命令
- 有时候可能会遇到开启vnc成功了，但是登陆不了的情况，可能是由于没有为所有用户开启的原因，可以尝试以下命令：
```objc
  sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/Contents/Resources/kickstart -activate -configure -access -off -restart -agent -privs -all -allowAccessFor -allUsers
``` 
- 使用以下命令关闭共享：
```objc
  sudo /System/Library/CoreServices/RemoteManagement/ARDAgent.app/  Contents/Resources/kickstart -deactivate -configure -access -off
```

#### ubuntu和centos安装wget
- ubuntu下安装
```objc
  sudo apt-get update  
  sudo apt-get install wget  
  wget --version
```

- centos下安装
```objc
  yum install wget
```

#### MacOS下查看jdk版本
```objc
  java -version
  //查看不同jdk安装位置
  /usr/libexec/java_home -V
```

#### 逆向
- 覆盖iPhone的RevealServer文件路径
```objc
  /Library/Frameworks/RevealServer.framework
  执行以下命令
  scp /Applications/Reveal.app/Contents/SharedSupport/iOS-   Libraries/RevealServer.framework/RevealServer root@192.168.1.12:/Library/Frameworks/RevealServer.framework
RevealServer                                  100% 9100KB  10.3MB/s   00:00
```

#### UIKit Mach-O文件的路径
```objc
  '/System/Library/Frameworks/UIKit.framework/UIKit'
```

#### 从iPhone拷贝UIKit.framework到电脑桌面
```objc
  scp -r root@192.168.1.12:/System/Library/Frameworks/   UIKit.framework ~/Desktop
```
     
     
   

## Link 
- [Markdown指南中文版](https://www.markdown.xyz/basic-syntax/)


