 Block  的实质究竟是什么呢？类型？变量？还是什么黑科技？
 Blocks 是 **带有局部变量的匿名函数**
# 1.1 Blocks 由 OC 转 C++ 源码方法
1. 在项目中添加 blocks.m 文件，并写好 block 的相关代码。
2. 打开「终端」，执行 `cd XXX/XXX` 命令，其中 `XXX/XXX` 为 block.m 所在的目录。
3. 继续执行`clang -rewrite-objc block.m`
4. 执行完命令之后，block.m 所在目录下就会生成一个 block.cpp 文件，这就是我们需要的 block 相关的 C++ 源码。
```c++
1. `/* 包含 Block 实际函数指针的结构体 */`
2. `struct __block_impl {`
3. `void *isa;`
4. `int Flags;`               
5. `int Reserved;        // 今后版本升级所需的区域大小`
6. `void *FuncPtr;      // 函数指针`
7. `};`

9. `/* Block 结构体 */`
10. `struct __main_block_impl_0 {`
11. `// impl：Block 的实际函数指针，指向包含 Block 主体部分的 __main_block_func_0 结构体`
12.     `struct __block_impl impl;`
13.     `// Desc：Desc 指针，指向包含 Block 附加信息的 __main_block_desc_0（） 结构体`
14.     `struct __main_block_desc_0* Desc;`
15.     `// __main_block_impl_0：Block 构造函数`
16.     `__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {`
17. `impl.isa = &_NSConcreteStackBlock;`
18.         `impl.Flags = flags;`
19.         `impl.FuncPtr = fp;`
20.         `Desc = desc;`
21.     `}`
22. `};`

24. `/* Block 主体部分结构体 */`
25. `static void __main_block_func_0(struct __main_block_impl_0 *__cself) {`
26. `printf("myBlock\n");`
27. `}`

29. `/* Block 附加信息结构体：包含今后版本升级所需区域大小，Block 的大小*/`
30. `static struct __main_block_desc_0 {`
31. `size_t reserved;        // 今后版本升级所需区域大小`
32. `size_t Block_size;    // Block 大小`
33. `} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};`

35. `/* main 函数 */`
36. `int main () {`
37.     `void (*myBlock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));`
38.     `((void (*)(__block_impl *))((__block_impl *)myBlock)->FuncPtr)((__block_impl *)myBlock);`

40.     `return 0;`
41. `}`
```
## Block 结构体

我们先来看看 `__main_block_impl_0` 结构体（ Block 结构体）
```c++
1. `/* Block 结构体 */`
2. `struct __main_block_impl_0 {`
3.     `// impl：Block 的实际函数指针，指向包含 Block 主体部分的 __main_block_func_0 结构体`
4.     `struct __block_impl impl;`
5.     `// Desc：Desc 指针，指向包含 Block 附加信息的 __main_block_desc_0（） 结构体`
6.     `struct __main_block_desc_0* Desc;`
7.     `// __main_block_impl_0：Block 构造函数`
8. `__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {`
9.         `impl.isa = &_NSConcreteStackBlock;`
10.         `impl.Flags = flags;`
11.         `impl.FuncPtr = fp;`
12.         `Desc = desc;`
13.     `}`
14. `};`
```
从上边我们可以看出，`__main_block_impl_0` 结构体（Block 结构体）包含了三个部分：

1. 成员变量 `impl`;
2. 成员变量 `Desc` 指针;  
 3. `__main_block_impl_0` 构造函数。
## `struct __block_impl impl` 说明
第一部分 `impl` 是 `__block_impl` 结构体类型的成员变量。`__block_impl` 包含了 Block 实际函数指针 `FuncPtr`，`FuncPtr` 指针指向 Block 的主体部分，也就是 Block 对应 OC 代码中的 `^{ printf("myBlock\n"); };` 部分。还包含了标志位 `Flags`，今后版本升级所需的区域大小  `Reserved`，`__block_impl` 结构体的实例指针 `isa`。
 