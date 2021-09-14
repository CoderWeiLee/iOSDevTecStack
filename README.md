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



## Shell
#### ssh登录远程服务器
```objc
ssh -p 188 root@171.17.12.80 //step 1
/*Enter Password */          //step 2
```
> -p 188指定端口号

## Link 
- [Markdown指南中文版](https://www.markdown.xyz/basic-syntax/)


