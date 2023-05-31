#### 方法的本质

方法的本质其实就是`objc_msgSend`消息发送，objc_msgSend的参数有

- 消息接收者
- 消息主体（SEL + 参数） 我们可以通过下面的方法验证，在main函数中，写入以下代码
```c
LGPerson *person = [LGPerson alloc]; 
[person sayNB]; 
[person say:@"NB"];

```
生成cpp文件，可以看到底层代码
```c
LGPerson *person = ((LGPerson *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("LGPerson"), sel_registerName("alloc")); 
((void (*)(id, SEL))(void *)objc_msgSend)((id)person, sel_registerName("sayNB")); 
((void (*)(id, SEL, NSString *))(void *)objc_msgSend)((id)person, sel_registerName("say:"), (NSString *)&__NSConstantStringImpl__var_folders_jl_d06jlfkj2ws74_5g45kms07m0000gn_T_main_d8842a_mi_3);

```
- sel_registerName为@selector()的底层实现
- 如果子类中没有方法，会向父类查找即调用`objc_msgSendSuper`，向父类发送消息

发送消息的主要流程如下：

- 快速查找：`objc_msgSend`查找`cache_t`缓存消息
- 慢速查找：递归自己和父类查找方法`lookUpImpOrForward`
- 查找不到消息，进行动态方法解析：`resolveInstanceMethod`
- 消息快速转发：`forwardingTargetForSeletor`
- 消息慢速转发：消息签名`methodSignatureForSelector`和分发`forwardInvocation`
- 最终仍为找到消息：程序crash，报经典错误消息`unrecognized selector sent to instance xxx`

#### SEL、IMP及两者关系

- SEL是方法编号，即方法名称，在dyld加载镜像时，通过read_image方法加载到内存的表中了。
- IMP是函数实现指针，找IMP就是找函数的过程
- 两者关系：sel相当于书本的目录标题，imp就是书本的页码。查找具体的函数就是想看这本书里面具体篇章的内容：
    - 我们首先要找到想看的标题们也就是title-sel
    - 然后根据目录找到对应的页码-imp
    - 找到具体的内容-方法的具体实现

#### 扩展

方法的快速查找流程是用汇编实现的，原因在于

- 快速查找流程，即：方法缓存查找，目的就是提升效率。使用汇编代码最接近机器语言，可以最大程度优化存储空间与执行时间
- 汇编代码对于动态参数、可变参数有更好的支持

二分查找法：

- 查找过程：表中方法编号按升序排列，将表中间位置记录的方法编号与将要查找的方法编号比较，如果两者相等，则查找成功；否则利用中间位置记录将表分成前、后两个子表，如果查找的方法编号大于中间位置记录的方法编号，则进一步查找后一子表，否则进一步查找前一子表。重复以上过程，直到找到满足条件的记录，此时查找成功。或直到子表不存在为止，此时查找不成功。

# Method IMP 概念介绍 

OC是消息转发机制，代码在编译的时候会生产Runtime中间代码，运行的时候执行Runtime代码，我们也可以动态的添加Runtime代码。
这篇之前讲过了如何创建类和Runtime中的属性，今天主要说一下关于Runtime的方法。

首先还要说一下Runtime类的结构体：
```c


struct objc_class {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class _Nullable super_class                              OBJC2_UNAVAILABLE;
    const char * _Nonnull name                               OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list *_Nullable ivars                  OBJC2_UNAVAILABLE;
    struct objc_method_list *_Nullable *_Nullable methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache *_Nonnull cache                       OBJC2_UNAVAILABLE;
    struct objc_protocol_list *_Nullable protocols          OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
```


然后我们可以看到：

  ```c
  struct objc_method_list * _Nullable * _Nullable methodLists                    OBJC2_UNAVAILABLE;
```


继续分析下 objc_method_list：
```c
struct objc_method_list {
    struct objc_method_list * _Nullable obsolete             OBJC2_UNAVAILABLE;

    int method_count                                         OBJC2_UNAVAILABLE;
#ifdef __LP64__
    int space                                                OBJC2_UNAVAILABLE;
#endif
    /* variable length structure */
    struct objc_method method_list[1]                        OBJC2_UNAVAILABLE;
}          
```

我们可以从上面的结构体上面看出  objc_method_list是 objc_method 结构体。


那么继续找 objc_method结构体 看看是否有IMP：
```c

struct objc_method {
    SEL _Nonnull method_name                                 OBJC2_UNAVAILABLE;
    char * _Nullable method_types                            OBJC2_UNAVAILABLE;
    IMP _Nonnull method_imp                                  OBJC2_UNAVAILABLE;
}      
```


分析：

看了根据上面的关系网，我们知道在OC的类中，所有的方法在Runtime的时候都会变成 objc_method，IMP是一个函数指针，指向实现这个函数的方法。然后这些方法都被放到objc_method_list这个结构体里。
## Runtime 关于Method IMP 的API介绍
```c
/* Working with Methods */

/** 
 * Returns the name of a method.
 * 
 * @param m The method to inspect.
 * 
 * @return A pointer of type SEL.
 * 
 * @note To get the method name as a C string, call \c sel_getName(method_getName(method)).
 */
OBJC_EXPORT SEL_Nonnull
method_getName(Method _Nonnull m) 
    OBJC_AVAILABLE(10.5,2.0,9.0,1.0,2.0);

/** 
 * Returns the implementation of a method.
 * 
 * @param m The method to inspect.
 * 
 * @return A function pointer of type IMP.
 */
OBJC_EXPORT IMP_Nonnull
method_getImplementation(Method _Nonnull m) 
    OBJC_AVAILABLE(10.5,2.0,9.0,1.0,2.0);

/** 
 * Returns a string describing a method's parameter and return types.
 * 
 * @param m The method to inspect.
 * 
 * @return A C string. The string may be \c NULL.
 */
OBJC_EXPORT const char * _Nullable
method_getTypeEncoding(Method _Nonnull m) 
    OBJC_AVAILABLE(10.5,2.0,9.0,1.0,2.0);

/** 
 * Returns the number of arguments accepted by a method.
 * 
 * @param m A pointer to a \c Method data structure. Pass the method in question.
 * 
 * @return An integer containing the number of arguments accepted by the given method.
 */
OBJC_EXPORT unsignedint
method_getNumberOfArguments(Method _Nonnull m)
    OBJC_AVAILABLE(10.0,2.0,9.0,1.0,2.0);

/** 
 * Returns a string describing a method's return type.
 * 
 * @param m The method to inspect.
 * 
 * @return A C string describing the return type. You must free the string with \c free().
 */
OBJC_EXPORT char * _Nonnull
method_copyReturnType(Method _Nonnull m) 
    OBJC_AVAILABLE(10.5,2.0,9.0,1.0,2.0);

/** 
 * Returns a string describing a single parameter type of a method.
 * 
 * @param m The method to inspect.
 * @param index The index of the parameter to inspect.
 * 
 * @return A C string describing the type of the parameter at index \e index, or \c NULL
 *  if method has no parameter index \e index. You must free the string with \c free().
 */
OBJC_EXPORT char * _Nullable
method_copyArgumentType(Method _Nonnull m, unsigned int index) 
    OBJC_AVAILABLE(10.5,2.0,9.0,1.0,2.0);

/** 
 * Returns by reference a string describing a method's return type.
 * 
 * @param m The method you want to inquire about. 
 * @param dst The reference string to store the description.
 * @param dst_len The maximum number of characters that can be stored in \e dst.
 *
 * @note The method's return type string is copied to \e dst.
 *  \e dst is filled as if \c strncpy(dst, parameter_type, dst_len) were called.
 */
OBJC_EXPORT void
method_getReturnType(Method _Nonnull m, char * _Nonnull dst, size_t dst_len) 
    OBJC_AVAILABLE(10.5,2.0,9.0,1.0,2.0);

/** 
 * Returns by reference a string describing a single parameter type of a method.
 * 
 * @param m The method you want to inquire about. 
 * @param index The index of the parameter you want to inquire about.
 * @param dst The reference string to store the description.
 * @param dst_len The maximum number of characters that can be stored in \e dst.
 * 
 * @note The parameter type string is copied to \e dst. \e dst is filled as if \c strncpy(dst, parameter_type, dst_len) 
 *  were called. If the method contains no parameter with that index, \e dst is filled as
 *  if \c strncpy(dst, "", dst_len) were called.
 */
OBJC_EXPORT void
method_getArgumentType(Method _Nonnull m, unsigned int index, 
                       char * _Nullable dst, size_t dst_len) 
    OBJC_AVAILABLE(10.5,2.0,9.0,1.0,2.0);

OBJC_EXPORT struct objc_method_description *_Nonnull
method_getDescription(Method _Nonnull m) 
    OBJC_AVAILABLE(10.5,2.0,9.0,1.0,2.0);

/** 
 * Sets the implementation of a method.
 * 
 * @param m The method for which to set an implementation.
 * @param imp The implemention to set to this method.
 * 
 * @return The previous implementation of the method.
 */
OBJC_EXPORT IMP_Nonnull
method_setImplementation(Method _Nonnull m, IMP _Nonnull imp) 
    OBJC_AVAILABLE(10.5,2.0,9.0,1.0,2.0);

/** 
 * Exchanges the implementations of two methods.
 * 
 * @param m1 Method to exchange with second method.
 * @param m2 Method to exchange with first method.
 * 
 * @note This is an atomic version of the following:
 *  \code 
 *  IMP imp1 = method_getImplementation(m1);
 *  IMP imp2 = method_getImplementation(m2);
 *  method_setImplementation(m1, imp2);
 *  method_setImplementation(m2, imp1);
 *  \endcode
 */
OBJC_EXPORT void
method_exchangeImplementations(Method _Null_unspecified /* _Nonnull */ m1, Method _Null_unspecified /* _Nonnull */ m2) 
    OBJC_AVAILABLE(10.5,2.0,9.0,1.0,2.0);
```

## Runtime 关于Method IMP 的应用示例

Demo1：
     获取方法名字和获取方法的IMP 。 从objc_method结构体可知，method里有SEL 方法名 和IMP 实现方法的函数指针组成。所以我们可以从方法里拿到他的属性，然后玩一玩，下面就是针对这两个方法的小应用。

通过下面示例我们可以明白：Method SEL IMP 外在联系

method_getName(Method _Nonnull m) 
method_getImplementation(Method _Nonnull m) 
```objective-c
- (void)viewDidLoad {
    [super viewDidLoad];
//首先我们早类里面找到该方法
    Method ori_Method = class_getInstanceMethod([self class], @selector(myMethod:));
#if 0
    //获取方法名 SEL
    SEL oriMethodName = method_getName(ori_Method);
    //根据方法名获取函数指针
    IMP myMethodImp  =  [self methodForSelector:oriMethodName];
#else
    //直接根据方法名获取函数指针  等同于IF 0
    IMP myMethodImp  =  method_getImplementation(ori_Method);
#endif
   
 //在该类中添加方法。
#if 0
    class_addMethod([self class], @selector(testMethod:), myMethodImp, method_getTypeEncoding(ori_Method));
#else
    class_addMethod([self class], @selector(testMethod:), myMethodImp, "V@:@");
#endif
    //在执行方法。
    [self performSelector:@selector(testMethod:) withObject:@"7777"];
}
 
-(void)myMethod:(NSString *)myValue{
    NSLog(@"myMethod:%@",myValue);
}

```

> 打印结果： 
**2017-07-20 10:52:26.581146+0800 zyTest[8351:416437] myMethod:7777**

Demo2：

中间关于获得方法参数，参数个数，参数返回类型就不赘述了，示例中有的会用到。主要介绍下面几个方法：

上面是GET IMP，那肯定可以设置方法的IMP
method_setImplementation(Method _Nonnull m, IMP _Nonnull imp) 
方法交换
method_exchangeImplementations(Method _Null_unspecified /* _Nonnull */ m1, Method _Null_unspecified /* _Nonnull */ m2) 

```objective-c
//方法交换
-(void)demo2{
    Method method1 = class_getInstanceMethod([self class], @selector(exchangeMethod1:));
    Method method2 = class_getInstanceMethod([self class], @selector(exchangeMethod2:));
    Method method3 = class_getInstanceMethod([self class], @selector(exchangeMethod3:));
    Method method4 = class_getInstanceMethod([self class], @selector(exchangeMethod4:));
    //方法替换   替换SEL 的IMP实现
    class_replaceMethod([self class], @selector(exchangeMethod1:), method_getImplementation(method3), method_getTypeEncoding(method3));
    //和class_replaceMethod 类似，替换method 的结构题IMP指针
    method_setImplementation(method4, method_getImplementation(method2));
    //方法交换
    method_exchangeImplementations(method1, method2);
    //猜一猜打印的什么
    [self performSelector:method_getName(method1) withObject:@"Runtime Method Demo1" afterDelay:0.0];
    [self exchangeMethod2:@"Runtime Method Demo2"];
    [self exchangeMethod3:@"Runtime Method Demo3"];
    [self exchangeMethod4:@"Runtime Method Demo4"];
}
 
-(void)exchangeMethod1:(id)str{
    NSLog(@"exchangeMethod1:%@",str);
}
 
-(void)exchangeMethod2:(id)num{
    NSLog(@"exchangeMethod2:%@",num);
}
 
-(void)exchangeMethod3:(id)f{
    NSLog(@"exchangeMethod3:%@",f);
}
 
-(void)exchangeMethod4:(id)w{
    NSLog(@"exchangeMethod4:%@",w);
}

```

