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