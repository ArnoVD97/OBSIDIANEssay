1. isa指针：isa指针是一个指向对象所属类或元类的指针。它决定了对象可以调用的方法和属性。isa指针在对象的结构中存在，并且在运行时会被自动设置。
2. isa 指针，表示这个对象是一个什么类。而 Class 类型， 也就是 struct objc_class * ，这是苹果在下面的注释中写到的。这说明类本身也是一个对象。在类对象中的 isa 指向的类叫做“元类”，类方法就定义在元类中。总的来说就是，一个类可以有很多的实例，这些实例有着唯一的一个类对象，而这个类对象也有着唯一的一个元类。
3. ![[Pasted image 20230529163548.png]]
在Objective-C中，每个对象都有一个isa指针，它指向该对象所属的类或元类。isa指针决定了对象可以调用的方法和属性。通过isa指针，Objective-C运行时可以在运行时动态地确定对象所属的类，并在该类或其父类中查找对应的方法实现。
下面是一些示例代码来说明isa指针的作用：
```c
@interface MyClass : NSObject
- (void)myMethod;
@end

@implementation MyClass
- (void)myMethod {
    NSLog(@"MyClass's myMethod");
}
@end

int main() {
    MyClass *myObject = [[MyClass alloc] init];
    [myObject myMethod];
    return 0;
}

```
在上面的示例中，创建了一个名为`MyClass`的类，它继承自`NSObject`。`MyClass`类中定义了一个名为`myMethod`的方法。

当我们创建一个`MyClass`对象并调用`myMethod`方法时，实际上发生了以下过程：

1. 分配内存并初始化`MyClass`对象。
2. 运行时为该对象设置isa指针，使其指向`MyClass`的类对象。
3. 在`myObject`上调用`myMethod`方法时，运行时首先通过isa指针找到`MyClass`的类对象。
4. 运行时在类对象中查找名为`myMethod`的方法实现并执行。

通过这个过程，我们可以看到isa指针在动态确定对象所属的类的过程中起到了关键作用。它使得我们可以在运行时根据对象的实际类型来调用适当的方法。
# isa，类对象，元类对象
OC的对象及其alloc和init。验证了OC对象底层是结构体，然后在alloc的方法调用栈的最后一个关键方法creatinstance中，有一个创建isa指针的方法。这篇，我们就先聊一聊isa指针。
我们知道，OC中的绝大部分对象，都是继承自NSObject(目前我已知的只有一个 _**NSProxy**_ 类不继承自NSObject，它跟NSObject一样，是基类)。按住command键，跳进NSObject的头文件，就能看到下面的代码：
```c
@interface NSObject <NSObject> {
    #pragma clang diagnostic push 
    #pragma clang diagnostic ignored "-Wobjc-interface-ivars"
    Class isa  OBJC_ISA_AVAILABILITY;
    #pragma clang diagnostic pop
}
```
这就是NSObject的声明了，它只有一个isa指针，万物继承自NSObject，就有了自己的isa指针了。那这个isa到底有什么用呢？它究竟是个啥东西？
## 位域
在说isa之前，我们先了解一下[[共用体]]和[[位域]]这种技术。

在armv7、armv7s时代，由于是32位操作系统，苹果并没有对指针进行优化，实例对象的isa是直接指向类对象的。进入到arm64架构时代后isa占用8个字节，共计64位，如果只是单纯存一个指针太浪费，所以，苹果通过共用体技术，充分的让这64位存储了非常多的信息，不用再额外的开辟空间来存储，减少了内存开支，减少了内存的操作。下面我们具体看一下isa的结构。
## isa的定义
```c
union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }
    Class cls;
    uintptr_t bits;
#if defined(ISA_BITFIELD)
    struct{
        ISA_BITFIELD;  // defined in isa.h  说明在isa.h中
    };
#endif
};

```
里面有个ISA_BITFIELD，再看这个是怎么定义的：
```c
# if __arm64__
#   define ISA_MASK        0x0000000ffffffff8ULL
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL
#   define ISA_BITFIELD                                                      \
      uintptr_t nonpointer        : 1;          //表明这个属性占一位，且从低位开始\
      uintptr_t has_assoc         : 1;          // 同上                       \
      uintptr_t has_cxx_dtor      : 1;                                       \
      uintptr_t shiftcls          : 33; /*MACH_VM_MAX_ADDRESS 0x1000000000*/ \
      uintptr_t magic             : 6;                                       \
      uintptr_t weakly_referenced : 1;                                       \
      uintptr_t deallocating      : 1;                                       \
      uintptr_t has_sidetable_rc  : 1;                                       \
      uintptr_t extra_rc          : 19
#   define RC_ONE   (1ULL<<45)
#   define RC_HALF  (1ULL<<18)

```
## isa union共用体中各个位的含义
- **nonpointer** 指示位，表示是否对isa指针开启指针优化，值为0表示纯isa指针 1表示不止是类对象的地址，isa中还包含了类信息、对象的引用计数等
    
- **has_assoc**: 关联对象标志位，0没有，1存在（KVO的实现原理）
    
- **has_cxx_dtor** :该对象是否有C++或者Objc的析构器，如果有析构函数，则需要做析构逻辑，如果没有，则可以更快的释放对象
    
- **shiftcls**: 实际存储类指针的值。开启指针优化的情况下，在arm64架构中有33位用来存储类指针，也是基于此，不同的[架构](https://link.juejin.cn/?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2Fc155c06185cb "https://www.jianshu.com/p/c155c06185cb")中，对于类指针的读取需要不同的操作，在32位时期，由于，由于未进行isa优化，isa是直接指向类对象的，在64位之后，需要用isa指针与上掩码(ISA_MASK)才能取到指针，这个指针指向类对象。
    
- **magic** :用于调试器判断当前对象是真的对象还是没有初始化的空间
    
- **weakly_refrenced**: 对象是否被指向或者曾经指向一个ARC的弱变量，没有弱引用的对象，可以更快的释放
    
- **dellocating** :标志对象是否正在释放对象。ing表示进行时
    
- **has_sidetable_rc**:当对象引用计数大于0时，则需要借用该变量存储进位。weak的实现原理，就是使用sidetables。后续讲到内存管理的时候会对此进行详细的讲解。
    
- **extra_rc**:当表示该对象的引用计数值，实际上是引用计数值减1.例如，如果对象的引用计数为10，那么extra_rc为9，如果引用计数大于10，则需要使用到上面的has_sidetable_rc。
    

用64位存储了这么多的信息，相当的高效！！我当初知道这个点的时候，叹为观止！苹果真的把这种细节这种设计做到了极致。苹果的工程师们真他娘的都是人才，各个身怀绝技（也可能我作为井底之蛙，没有了解过这个技术）。

好，介绍过isa之后，接下来就该是实例对象、类对象、和元类对象了。

在arm64架构之后，isa通过shiftcls来指向类对象的。类对象也是个对象，也有isa指针，它的isa中的shiftcls是指向元类对象的。元类对象也有isa，它的shiftcls都指向根类的元类对象。 网上有一张非常经典的图来展示了对象和元类对象的关系（这幅图将贯穿大部分关于底层原理的博客内容）来看一下：

  