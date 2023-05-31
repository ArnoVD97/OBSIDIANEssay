[[RunTime-One]]
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
## 2.1 Method Swizzling 简单使用
在当前类的 `+ (void)load;` 方法中增加 Method Swizzling 操作，交换 `- (void)originalFunction;` 和 `- (void)swizzledFunction;` 的方法实现。
```objective-c
1. `#import "ViewController.h"`
2. `#import <objc/runtime.h>`

4. `@interface ViewController ()`

6. `@end`

8. `@implementation ViewController`

10. `- (void)viewDidLoad {`
11. `[super viewDidLoad];`

13. `[self SwizzlingMethod];`
14. `[self originalFunction];`
15. `[self swizzledFunction];`
16. `}`

18. `// 交换 原方法 和 替换方法 的方法实现`
19. `- (void)SwizzlingMethod {`
20. `// 当前类`
21. `Class class = [self class];`

23. `// 原方法名 和 替换方法名`
24. `SEL originalSelector = @selector(originalFunction);`
25. `SEL swizzledSelector = @selector(swizzledFunction);`

27. `// 原方法结构体 和 替换方法结构体`
28. `Method originalMethod = class_getInstanceMethod(class, originalSelector);`
29. `Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);`

31. `// 调用交换两个方法的实现`
32. `method_exchangeImplementations(originalMethod, swizzledMethod);`
33. `}`

35. `// 原始方法`
36. `- (void)originalFunction {`
37. `NSLog(@"originalFunction");`
38. `}`

40. `// 替换方法`
41. `- (void)swizzledFunction {`
42. `NSLog(@"swizzledFunction");`
43. `}`

45. `@end`
```
> 打印结果：  
2019-07-12 09:59:19.672349+0800 Runtime-MethodSwizzling[91009:30112833] swizzledFunction  
2019-07-12 09:59:20.414930+0800 Runtime-MethodSwizzling[91009:30112833] originalFunction

可以看出两者方法成功进行了交换。
刚才我们简单演示了如何在当前类中如何进行 Method Swizzling 操作。但一般日常开发中，并不是直接在原有类中进行 Method Swizzling 操作。更多的是为当前类添加一个分类，然后在分类中进行 Method Swizzling 操作。另外真正使用会比上边写的考虑东西要多一点，要复杂一些。

在日常使用 Method Swizzling 的过程中，有几种很常用的方案，具体情况如下。
## 2.1 Method Swizzling 方案 A
> 在该类的分类中添加 Method Swizzling 交换方法，用普通方式

这种方式在开发中应用最多的。但是还是要注意一些事项，我会在接下来的 **3. Method Swizzling 使用注意** 进行详细说明。
```objective-c
1. `@implementation UIViewController (Swizzling)`

3. `// 交换 原方法 和 替换方法 的方法实现`
4. `+ (void)load {`

6. `static dispatch_once_t onceToken;`
7. `dispatch_once(&onceToken, ^{`
8. `// 当前类`
9. `Class class = [self class];`

11. `// 原方法名 和 替换方法名`
12. `SEL originalSelector = @selector(originalFunction);`
13. `SEL swizzledSelector = @selector(swizzledFunction);`

15. `// 原方法结构体 和 替换方法结构体`
16. `Method originalMethod = class_getInstanceMethod(class, originalSelector);`
17. `Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);`

19. `/* 如果当前类没有 原方法的 IMP，说明在从父类继承过来的方法实现，`
20. `* 需要在当前类中添加一个 originalSelector 方法，`
21. `* 但是用 替换方法 swizzledMethod 去实现它`
22. `*/`
23. `BOOL didAddMethod = class_addMethod(class,`
24. `originalSelector,`
25. `method_getImplementation(swizzledMethod),`
26. `method_getTypeEncoding(swizzledMethod));`

28. `if (didAddMethod) {`
29. `// 原方法的 IMP 添加成功后，修改 替换方法的 IMP 为 原始方法的 IMP`
30. `class_replaceMethod(class,`
31. `swizzledSelector,`
32. `method_getImplementation(originalMethod),`
33. `method_getTypeEncoding(originalMethod));`
34. `} else {`
35. `// 添加失败（说明已包含原方法的 IMP），调用交换两个方法的实现`
36. `method_exchangeImplementations(originalMethod, swizzledMethod);`
37. `}`
38. `});`
39. `}`

41. `// 原始方法`
42. `- (void)originalFunction {`
43. `NSLog(@"originalFunction");`
44. `}`

46. `// 替换方法`
47. `- (void)swizzledFunction {`
48. `NSLog(@"swizzledFunction");`
49. `}`

51. `@end`
```
## 2.2 Method Swizzling 方案 B
> 在该类的分类中添加 Method Swizzling 交换方法，但是使用函数指针的方式。

方案 B 和方案 A 的最大不同之处在于使用了函数指针的方式，使用函数指针最大的好处是可以有效避免命名错误。
```objective-c
1. `#import "UIViewController+PointerSwizzling.h"`
2. `#import <objc/runtime.h>`

4. `typedef IMP *IMPPointer;`

6. `// 交换方法函数`
7. `static void MethodSwizzle(id self, SEL _cmd, id arg1);`
8. `// 原始方法函数指针`
9. `static void (*MethodOriginal)(id self, SEL _cmd, id arg1);`

11. `// 交换方法函数`
12. `static void MethodSwizzle(id self, SEL _cmd, id arg1) {`

14. `// 在这里添加 交换方法的相关代码`
15. `NSLog(@"swizzledFunc");`

17. `MethodOriginal(self, _cmd, arg1);`
18. `}`

20. `BOOL class_swizzleMethodAndStore(Class class, SEL original, IMP replacement, IMPPointer store) {`
21. `IMP imp = NULL;`
22. `Method method = class_getInstanceMethod(class, original);`
23. `if (method) {`
24. `const char *type = method_getTypeEncoding(method);`
25. `imp = class_replaceMethod(class, original, replacement, type);`
26. `if (!imp) {`
27. `imp = method_getImplementation(method);`
28. `}`
29. `}`
30. `if (imp && store) { *store = imp; }`
31. `return (imp != NULL);`
32. `}`

34. `@implementation UIViewController (PointerSwizzling)`

36. `+ (void)load {`
37. `static dispatch_once_t onceToken;`
38. `dispatch_once(&onceToken, ^{`
39. `[self swizzle:@selector(originalFunc) with:(IMP)MethodSwizzle store:(IMP *)&MethodOriginal];`
40. `});`
41. `}`

43. `+ (BOOL)swizzle:(SEL)original with:(IMP)replacement store:(IMPPointer)store {`
44. `return class_swizzleMethodAndStore(self, original, replacement, store);`
45. `}`

47. `// 原始方法`
48. `- (void)originalFunc {`
49. `NSLog(@"originalFunc");`
50. `}`

52. `@end`
```
## 2.3 Method Swizzling 方案 C
>在其他类中添加 Method Swizzling 交换方法

这种情况一般用的不多，最出名的就是 AFNetworking 中的_AFURLSessionTaskSwizzling 私有类。_AFURLSessionTaskSwizzling 主要解决了 iOS7 和 iOS8 系统上 NSURLSession 差别的处理。让不同系统版本 NSURLSession 版本基本一致。
```objective-c
1. `static inline void af_swizzleSelector(Class theClass, SEL originalSelector, SEL swizzledSelector) {`
2. `Method originalMethod = class_getInstanceMethod(theClass, originalSelector);`
3. `Method swizzledMethod = class_getInstanceMethod(theClass, swizzledSelector);`
4. `method_exchangeImplementations(originalMethod, swizzledMethod);`
5. `}`

7. `static inline BOOL af_addMethod(Class theClass, SEL selector, Method method) {`
8. `return class_addMethod(theClass, selector, method_getImplementation(method), method_getTypeEncoding(method));`
9. `}`

11. `@interface _AFURLSessionTaskSwizzling : NSObject`

13. `@end`

15. `@implementation _AFURLSessionTaskSwizzling`

17. `+ (void)load {`
18. `if (NSClassFromString(@"NSURLSessionTask")) {`

20. `NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration ephemeralSessionConfiguration];`
21. `NSURLSession * session = [NSURLSession sessionWithConfiguration:configuration];`
22. `#pragma GCC diagnostic push`
23. `#pragma GCC diagnostic ignored "-Wnonnull"`
24. `NSURLSessionDataTask *localDataTask = [session dataTaskWithURL:nil];`
25. `#pragma clang diagnostic pop`
26. `IMP originalAFResumeIMP = method_getImplementation(class_getInstanceMethod([self class], @selector(af_resume)));`
27. `Class currentClass = [localDataTask class];`

29. `while (class_getInstanceMethod(currentClass, @selector(resume))) {`
30. `Class superClass = [currentClass superclass];`
31. `IMP classResumeIMP = method_getImplementation(class_getInstanceMethod(currentClass, @selector(resume)));`
32. `IMP superclassResumeIMP = method_getImplementation(class_getInstanceMethod(superClass, @selector(resume)));`
33. `if (classResumeIMP != superclassResumeIMP &&`
34. `originalAFResumeIMP != classResumeIMP) {`
35. `[self swizzleResumeAndSuspendMethodForClass:currentClass];`
36. `}`
37. `currentClass = [currentClass superclass];`
38. `}`

40. `[localDataTask cancel];`
41. `[session finishTasksAndInvalidate];`
42. `}`
43. `}`

45. `+ (void)swizzleResumeAndSuspendMethodForClass:(Class)theClass {`
46. `Method afResumeMethod = class_getInstanceMethod(self, @selector(af_resume));`
47. `Method afSuspendMethod = class_getInstanceMethod(self, @selector(af_suspend));`

49. `if (af_addMethod(theClass, @selector(af_resume), afResumeMethod)) {`
50. `af_swizzleSelector(theClass, @selector(resume), @selector(af_resume));`
51. `}`

53. `if (af_addMethod(theClass, @selector(af_suspend), afSuspendMethod)) {`
54. `af_swizzleSelector(theClass, @selector(suspend), @selector(af_suspend));`
55. `}`
56. `}`

58. `- (void)af_resume {`
59. `NSAssert([self respondsToSelector:@selector(state)], @"Does not respond to state");`
60. `NSURLSessionTaskState state = [self state];`
61. `[self af_resume];`

63. `if (state != NSURLSessionTaskStateRunning) {`
64. `[[NSNotificationCenter defaultCenter] postNotificationName:AFNSURLSessionTaskDidResumeNotification object:self];`
65. `}`
66. `}`

68. `- (void)af_suspend {`
69. `NSAssert([self respondsToSelector:@selector(state)], @"Does not respond to state");`
70. `NSURLSessionTaskState state = [self state];`
71. `[self af_suspend];`

73. `if (state != NSURLSessionTaskStateSuspended) {`
74. `[[NSNotificationCenter defaultCenter] postNotificationName:AFNSURLSessionTaskDidSuspendNotification object:self];`
75. `}`
76. `}`
```
## 2.4 Method Swizzling 方案 D
> 优秀的第三方框架：[JRSwizzle](https://link.jianshu.com/?t=https://github.com/rentzsch/jrswizzle) 和 [RSSwizzle](https://link.jianshu.com/?t=https://github.com/rabovik/RSSwizzle)

JRSwizzle 和 RSSwizzle 都是优秀的封装 Method Swizzling 的第三方框架。

1. **JRSwizzle** 尝试解决在不同平台和系统版本上的 Method Swizzling 与类继承关系的冲突。对各平台低版本系统兼容性较强。JRSwizzle 核心是用到了 `method_exchangeImplementations` 方法。在健壮性上先做了 `class_addMethod` 操作。
    
2. **RSSwizzle** 主要用到了 `class_replaceMethod` 方法，避免了子类的替换影响了父类。而且对交换方法过程加了锁，增强了线程安全。它用很复杂的方式解决了 **[What are the dangers of method swizzling in Objective-C？](https://stackoverflow.com/questions/5339276/what-are-the-dangers-of-method-swizzling-in-objective-c)** 中提到的问题。是一种更安全优雅的 Method Swizzling 解决方案。
 ### 总结：
 

在开发中我们通常使用方案 A，或者方案 D 中的第三方框架 RSSwizzle 来实现 Method Swizzling。在接下来 **3. Method Swizzling** 使用注意 中，我们还讲看到很多的注意事项。这些注意事项并不是为了吓退初学者，而是为了更好的使用 Method Swizzling 这一利器。而至于方案的选择，无论是选择哪种方案，我认为只有最适合项目的方案才是最佳方案。
