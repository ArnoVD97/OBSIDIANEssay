将代码转化为可执行程序，通常需要经过三个步骤：编译，连接，运行。不同的语言，这三个步骤的操作也有些不同
目前只看oc，动态语言，在编译阶段并不知道变量的具体类型，也不知道真正调用的哪个函数，只有在运行期才会检查数据类型，同时在运行时才会根据函数名查找具体的要调用的函数，这样在程序没运行的时候，我们不知道调用一个方法具体会发生什么
oc吧一些决定性的工作从编译链接阶段推迟到 `运行时阶段`，甚至可以在程序运行时候动态修改一个方法的实现，这也为大为流行的「热更新」提供了可能性。
RUNTIME实际是一个库，这个库使程序在运行的时候动态创建对象，检查对象，修改类和对象的方法
# 消息机制的基本原理
对象方法调用都是类似 `[receiver selector];` 的形式，其本质就是让对象在运行时发送消息的过程。

我们来看看方法调用 `[receiver selector];` 在「编译阶段」和「运行阶段」分别做了什么？
1. 编译阶段[receiver selector] 方法被编译器转化为
objc_msgSend(receiver, selector)不带参数
objc_msgSend(receiver, selector, org1, org2, ...)带参数
2. 运行时：消息接受者receiver寻找对应的selector。
通过对应的[[isa指针]]找到receiver的Class类
在Class类的method list方法列表中找到对应的selector
如果在Class中没找到这个selector，就继续在它的superClass类中寻找
3. 一旦在找到对应的selector方法，直接执行receiver对应的selector方法实现的[[IMP方法实现]]
4. 若找不到对应的selector，若没有实现消息被转发或者临时向receiver添加这个selector对应的实现方法，否则就会崩溃
在上述过程中涉及了好几个新的概念：`objc_msgSend`、`isa 指针`、`Class（类）`、`IMP（方法实现）` 等，下面我们来具体讲解一下各个概念的含义。
# Runtime 中的概念解析
## objc_msgSend

所有 Objective-C 方法调用在编译时都会转化为对 C 函数 `objc_msgSend` 的调用。`objc_msgSend(receiver，selector);` 是 `[receiver selector];` 对应的 C 函数。
## Class（类）

在 `objc/runtime.h` 中，`Class（类）` 被定义为指向 `objc_class 结构体` 的指针，`objc_class 结构体` 的数据结构如下：
```c#
1. `/// An opaque type that represents an Objective-C class.`
2. `typedef struct objc_class *Class;`

4. `struct objc_class {`
5. `Class _Nonnull isa; // objc_class 结构体的实例指针`

7. `#if !__OBJC2__`
8. `Class _Nullable super_class; // 指向父类的指针`
9. `const char * _Nonnull name; // 类的名字`
10. `long version; // 类的版本信息，默认为 0`
11. `long info; // 类的信息，供运行期使用的一些位标识`
12. `long instance_size; // 该类的实例变量大小;`
13. `struct objc_ivar_list * _Nullable ivars; // 该类的实例变量列表`
14. `struct objc_method_list * _Nullable * _Nullable methodLists; // 方法定义的列表`
15. `struct objc_cache * _Nonnull cache; // 方法缓存`
16. `struct objc_protocol_list * _Nullable protocols; // 遵守的协议列表`
17. `#endif`

19. `};`
```
从中可以看出，`objc_class 结构体` 定义了很多变量：自身的所有实例变量（ivars）、所有方法定义（methodLists）、遵守的协议列表（protocols）等。`objc_class 结构体` 存放的数据称为 **元数据（metadata）**。

`objc_class 结构体` 的第一个成员变量是 `isa 指针`，`isa 指针` 保存的是所属类的结构体的实例的指针，这里保存的就是 `objc_class 结构体`的实例指针，而实例换个名字就是 **对象**。换句话说，`Class（类）` 的本质其实就是一个对象，我们称之为 **类对象**。
## Object（对象）
接下来，我们再来看看 `objc/objc.h` 中关于 `Object（对象）` 的定义。  
`Object（对象）`被定义为 `objc_object 结构体`，其数据结构如下：
```c#
1. `/// Represents an instance of a class.`
2. `struct objc_object {`
3. `Class _Nonnull isa; // objc_object 结构体的实例指针`
4. `};`

6. `/// A pointer to an instance of a class.`
7. `typedef struct objc_object *id;`
```
这里的 `id` 被定义为一个指向 `objc_object 结构体` 的指针。从中可以看出 `objc_object 结构体` 只包含一个 `Class` 类型的 `isa 指针`。
```c
UIlabel *arno = [[UILabel alloc] init];
//我创建了一个UILabel对象，arno是一个对象内容是一个isa指针，指向同时创建的类对象，alloc为类对象开辟了一个空间，data存在arnoisa指向的类对象空间中，类对象里面也有一个isa指针，指向其元类，元类存储了类对象的方法和静态变量等信息。每个类对象在运行时都有一个对应的元类。
```
换句话说，一个 `Object（对象）`唯一保存的就是它所属 `Class（类）` 的地址。当我们对一个对象，进行方法调用时，比如 `[receiver selector];`，它会通过 `objc_object 结构体`的 `isa 指针` 去找对应的 `object_class 结构体`，然后在 `object_class 结构体` 的 `methodLists（方法列表）` 中找到我们调用的方法，然后执行。
## Meta Class
从上边我们看出， `对象（objc_object 结构体）`的 `isa 指针` 指向的是对应的 `类对象（object_class 结构体）`。那么 `类对象（object_class 结构体`）的 `isa 指针` 又指向什么呢？
`object_class 结构体` 的 `isa 指针` 实际上指向的的是 `类对象` 自身的 `Meta Class（元类）`。
那么什么是 `Meta Class（元类）`？
`Meta Class（元类）` 就是一个类对象所属的 **类**。一个对象所属的类叫做 `类对象`，而一个类对象所属的类就叫做 **元类**。
**当你给对象发送消息时，消息是在寻找这个对象的类的方法列表。  
当你给类发消息时，消息是在寻找这个类的元类的方法列表。**
对象 -> 类对象->元类
> Runtime 中把类对象所属类型就叫做 `Meta Class（元类）`，用于描述类对象本身所具有的特征，而在元类的 methodLists 中，保存了类的方法链表，即所谓的「类方法」。并且类对象中的 `isa 指针` 指向的就是元类。每个类对象有且仅有一个与之相关的元类。

在 **2. 消息机制的基本原理** 中我们讲解了 **对象方法的调用过程**，我们是通过对象的 `isa 指针` 找到 对应的 `Class（类）`；然后在 `Class（类）` 的 `method list（方法列表）` 中找对应的 `selector` 。
而 **类方法的调用过程** 和对象方法调用差不多，流程如下：

1. 通过类对象 `isa 指针` 找到所属的 `Meta Class（元类）`；
2. 在  `Meta Class（元类）` 的 `method list（方法列表）` 中找到对应的 `selector`;
3. 执行对应的 `selector`。
4. 下面看一个示例：

```objective-c
NSString *testString = [NSString stringWithFormat:@"%d,%s",3, "test"];
```

上边的示例中，`stringWithFormat:` 被发送给了 `NSString 类`，`NSString 类` 通过 `isa 指针` 找到 `NSString 元类`，然后在该元类的方法列表中找到对应的 `stringWithFormat:` 方法，然后执行该方法。
## 实例对象、类、元类之间的关系
上面，我们讲解了 **实例对象（Object）**、**类（Class）**、**Meta Class（元类）** 的基本概念，以及简单的指向关系。下面我们通过一张图来清晰地表示出这种关系。
![[Pasted image 20230529155240.png]]
我们先来看 `isa 指针`：

1. 水平方向上，每一级中的 `实例对象` 的 `isa 指针` 指向了对应的 `类对象`，而 `类对象` 的 `isa 指针` 指向了对应的 `元类`。而所有元类的 `isa 指针` 最终指向了 `NSObject 元类`，因此 `NSObject 元类` 也被称为 `根源类`。
2. 垂直方向上， `元类` 的 `isa 指针` 和 `父类元类` 的 `isa 指针` 都指向了 `根元类`。而 `根源类` 的 `isa 指针` 又指向了自己。

我们再来看 `父类指针`：

1. `类对象` 的  `父类指针` 指向了 `父类的类对象`，`父类的类对象` 又指向了 `根类的类对象`，`根类的类对象` 最终指向了 nil。
2. `元类` 的 `父类指针` 指向了 `父类对象的元类`。`父类对象的元类` 的 `父类指针`指向了 `根类对象的元类`，也就是 `根元类`。而 `根元类` 的 `父亲指针` 指向了 `根类对象`，最终指向了 nil。
## 方法
`object_class 结构体` 的 `methodLists（方法列表）`中存放的元素就是 `方法（Method）`。

先来看下 `objc/runtime.h` 中，表示 `方法（Method）` 的 `objc_method 结构体` 的数据结构：
```c
1. `/// An opaque type that represents a method in a class definition.`
2. `/// 代表类定义中一个方法的不透明类型`
3. `typedef struct objc_method *Method;`

5. `struct objc_method {`
6.     `SEL _Nonnull method_name;                    // 方法名`
7.     `char * _Nullable method_types;               // 方法类型`
8.     `IMP _Nonnull method_imp;                     // 方法实现`
9. `};`
```

可以看到，`objc_method 结构体` 中包含了 `方法名（method_name）`，`方法类型（method_types）` 和 `方法实现（method_imp）`。下面，我们来了解下这三个变量。
1. `SEL method_name; // 方法名`
```c
1. `/// An opaque type that represents a method selector.`
2. `typedef struct objc_selector *SEL;`
```


`SEL` 是一个指向 `objc_selector 结构体` 的指针，但是在 runtime 相关头文件中并没有找到明确的定义。不过，通过测试我们可以得出： `SEL` 只是一个保存方法名的字符串。
```c
1. `SEL sel = @selector(viewDidLoad);`
2. `NSLog(@"%s", sel); // 输出：viewDidLoad`
3. `SEL sel1 = @selector(test);`
4. `NSLog(@"%s", sel1); // 输出：test`
```
2. `IMP method_imp; // 方法实现`
```c
1. `/// A pointer to the function of a method implementation.`
2. `#if !OBJC_OLD_DISPATCH_PROTOTYPES`
3. `typedef void (*IMP)(void /* id, SEL, ... */ );`
4. `#else`
5. `typedef id _Nullable (*IMP)(id _Nonnull, SEL _Nonnull, ...);`
6. `#endif`
```
`IMP` 的实质是一个函数指针，所指向的就是方法的实现。`IMP`用来找到函数地址，然后执行函数。
3. `char * method_types; // 方法类型`

方法类型 `method_types` 是个字符串，用来存储方法的参数类型和返回值类型。
> 到这里， `Method` 的结构就已经很清楚了，`Method` 将 `SEL（方法名）` 和 `IMP（函数指针）` 关联起来，当对一个对象发送消息时，会通过给出的 `SEL（方法名）` 去找到  `IMP（函数指针）` ，然后执行。
# RunTime消息转发
在 **2. 消息机制的基本原理** 最后一步中我们提到：若找不到对应的 `selector`，必须消息被转发或者临时向 `recever` 添加这个 `selector` 对应的实现方法，否则就会发生崩溃。

当一个方法找不到的时候，Runtime 提供了 **消息动态解析**、**消息接受者重定向**、**消息重定向** 等三步处理消息，具体流程如下图所示：
![[Pasted image 20230531170631.png]]
## 4.1 消息动态解析
OC运行时调用+resolveInstanceMethod：或者+resolveClassMethod：，让你有机会提供一个函数实现。我们重写这两个方法，添加其他函数实现，并返回YES，那么运行时系统就会重新启动一次消息发送的过程。
主要用法：
```objective-c
1. `// 类方法未找到时调起，可以在此添加类方法实现`
2. `+ (BOOL)resolveClassMethod:(SEL)sel;`
3. `// 对象方法未找到时调起，可以在此对象方法实现`
4. `+ (BOOL)resolveInstanceMethod:(SEL)sel;`

6. `/**`
7. `* class_addMethod 向具有给定名称和实现的类中添加新方法`
8. `* @param cls 被添加方法的类`
9. `* @param name selector 方法名`
10. `* @param imp 实现方法的函数指针`
11. `* @param types imp 指向函数的返回值与参数类型`
12. `* @return 如果添加方法成功返回 YES，否则返回 NO`
13. `*/`
14. `BOOL class_addMethod(Class cls, SEL name, IMP imp,`
15. `const char * _Nullable types);`
```
举个例子
```c
1. `#import "ViewController.h"`
2. `#include "objc/runtime.h"`

4. `@interface ViewController ()`

6. `@end`

8. `@implementation ViewController`

10. `- (void)viewDidLoad {`
11. `[super viewDidLoad];`

13. `// 执行 fun 函数`
14. `[self performSelector:@selector(fun)];`
15. `}`

17. `// 重写 resolveInstanceMethod: 添加对象方法实现`
18. `+ (BOOL)resolveInstanceMethod:(SEL)sel {`
19. `if (sel == @selector(fun)) { // 如果是执行 fun 函数，就动态解析，指定新的 IMP`
20. `class_addMethod([self class], sel, (IMP)funMethod, "v@:");`
21. `return YES;`
22. `}`
23. `return [super resolveInstanceMethod:sel];`
24. `}`

26. `void funMethod(id obj, SEL _cmd) {`
27. `NSLog(@"funMethod"); //新的 fun 函数`
28. `}`

30. `@end`

```
> 打印结果：  
2019-06-12 10:25:39.848260+0800 runtime[14884:7977579] funMethod

从上边的例子中，我们可以看出，虽然我们没有实现 `fun` 方法，但是通过重写 `resolveInstanceMethod:` ，利用 `class_addMethod` 方法添加对象方法实现 `funMethod` 方法，并执行。从打印结果来看，成功调起了`funMethod` 方法。
我们注意到 class_addMethod 方法中的特殊参数 `v@:`，具体可参考官方文档中关于 [[Type Encodings]] 的说明：[传送门](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100)
## 4.2 消息接受者重定向
如果上一步中 `+resolveInstanceMethod:` 或者 `+resolveClassMethod:` 没有添加其他函数实现，运行时就会进行下一步：消息接受者重定向。

如果当前对象实现了 `-forwardingTargetForSelector:`，Runtime 就会调用这个方法，允许我们将消息的接受者转发给其他对象。
用到的方法：
```c
1. `// 重定向方法的消息接收者，返回一个类或实例对象`
2. `- (id)forwardingTargetForSelector:(SEL)aSelector;`
```
