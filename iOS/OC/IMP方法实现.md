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

Method IMP 概念介绍 

OC是消息转发机制，代码在编译的时候会生产Runtime中间代码，运行的时候执行Runtime代码，我们也可以动态的添加Runtime代码。
这篇之前讲过了如何创建类和Runtime中的属性，今天主要说一下关于Runtime的方法。

首先还要说一下Runtime类的结构体：

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


然后我们可以看到：

    struct objc_method_list * _Nullable * _Nullable methodLists                    OBJC2_UNAVAILABLE;



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
