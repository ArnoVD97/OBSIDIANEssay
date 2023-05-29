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

换句话说，一个 `Object（对象）`唯一保存的就是它所属 `Class（类）` 的地址。当我们对一个对象，进行方法调用时，比如 `[receiver selector];`，它会通过 `objc_object 结构体`的 `isa 指针` 去找对应的 `object_class 结构体`，然后在 `object_class 结构体` 的 `methodLists（方法列表）` 中找到我们调用的方法，然后执行。
## Meta Class
从上边我们看出， `对象（objc_object 结构体）`的 `isa 指针` 指向的是对应的 `类对象（object_class 结构体）`。那么 `类对象（object_class 结构体`）的 `isa 指针` 又指向什么呢？
`object_class 结构体` 的 `isa 指针` 实际上指向的的是 `类对象` 自身的 `Meta Class（元类）`。
那么什么是 `Meta Class（元类）`？
`Meta Class（元类）` 就是一个类对象所属的 **类**。一个对象所属的类叫做 `类对象`，而一个类对象所属的类就叫做 **元类**。
对象 -> 类对象->元类
> Runtime 中把类对象所属类型就叫做 `Meta Class（元类）`，用于描述类对象本身所具有的特征，而在元类的 methodLists 中，保存了类的方法链表，即所谓的「类方法」。并且类对象中的 `isa 指针` 指向的就是元类。每个类对象有且仅有一个与之相关的元类。

在 **2. 消息机制的基本原理** 中我们讲解了 **对象方法的调用过程**，我们是通过对象的 `isa 指针` 找到 对应的 `Class（类）`；然后在 `Class（类）` 的 `method list（方法列表）` 中找对应的 `selector` 。
而 **类方法的调用过程** 和对象方法调用差不多，流程如下：

1. 通过类对象 `isa 指针` 找到所属的 `Meta Class（元类）`；
2. 在  `Meta Class（元类）` 的 `method list（方法列表）` 中找到对应的 `selector`;
3. 执行对应的 `selector`。
4. 下面看一个示例：

```objec
NSString *testString = [NSString stringWithFormat:@"%d,%s",3, "test"];
```

上边的示例中，`stringWithFormat:` 被发送给了 `NSString 类`，`NSString 类` 通过 `isa 指针` 找到 `NSString 元类`，然后在该元类的方法列表中找到对应的 `stringWithFormat:` 方法，然后执行该方法。