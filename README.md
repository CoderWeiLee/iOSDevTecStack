# iOSDevTecStack
iOS开发者需要具备的技术栈各种知识点收集汇总整理，不断完善中，欢迎提issue/PR

## Swift
#### dynamic
- 被`@objc dynamic` 修饰的内容会具有动态性，比如调用方法会走`runtime`那一套流程

#### 实现只能被类遵守的协议
```objc
  protocol Runnable: Anyobject {}
  protocol Runnable: class {}
  @objc protocol Runnable {}
```
> @objc定义的协议，也可以被OC的类去遵守实现

#### 实现可选协议的方式
- 通过`Extension`提供协议中方法的默认实现
- ```objc
   @objc protocol Runnable {
     @objc optional func run1
     func run2
   }
```
> 其中，定义的run1就是可选的方法

