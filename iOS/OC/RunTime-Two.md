Runtime 在实际开发过程中，具体的应用场景。
这一篇我们来学习一下被称为 Runtime 运行时系统中最具争议的黑魔法：**Method Swizzling（动态方法交换）**
# 1. Method Swizzling（动态方法交换）简介

**Method Swizzling** 用于改变一个已经存在的 selector 实现。我们可以在程序运行时，通过改变 selector 所在 Class（类）的 method list（方法列表）的映射从而改变方法的调用。其实质就是交换两个方法的 IMP（方法实现）。

上一篇文章中我们知道：`Method（方法）`对应的是 `objc_method 结构体`；而 `objc_method 结构体` 中包含了 `SEL method_name（方法名）`、`IMP method_imp（方法实现）`。
```c
1. `// objc_method 结构体`
2. `typedef struct objc_method *Method;`

4. `struct objc_method {`
5. `SEL _Nonnull method_name; // 方法名`
6. `char * _Nullable method_types; // 方法类型`
7. `IMP _Nonnull method_imp; // 方法实现`
8. `};`
```
`Method（方法）`、`SEL（方法名）`、`IMP（方法实现）`三者的关系可以这样来表示：

在运行时，`Class（类）` 维护了一个 `method list（方法列表）` 来确定消息的正确发送。`method list（方法列表）` 存放的元素就是 `Method（方法）`。而 `Method（方法）` 中映射了一对键值对：`SEL（方法名）：IMP（方法实现）`。

Method swizzling 修改了 method list（方法列表），使得不同 `Method（方法）`中的键值对发生了交换。比如交换前两个键值对分别为 `SEL A : IMP A`、`SEL B : IMP B`，交换之后就变为了  `SEL A : IMP B`、`SEL B : IMP A`。如图所示：![[Pasted image 20230531213624.png]]
# 2. Method Swizzling 使用方法

假如当前类中有两个方法：`- (void)originalFunction;` 和 `- (void)swizzledFunction;`。如果我们想要交换两个方法的实现，从而实现调用 `- (void)originalFunction;` 方法实际上调用的是 `- (void)swizzledFunction;` 方法，而调用 `- (void)swizzledFunction;` 方法实际上调用的是 `- (void)originalFunction;` 方法的效果。那么我们需要像下边代码一样来实现。