为了帮助运行时系统，编译器在字符串中对每个方法的返回和参数类型进行编码，并将字符串与方法选择器相关联。它使用的编码方案在其他上下文中也很有用，因此通过@encode()编译器指令公开提供。当给定类型规范时，@encode()返回对该类型进行编码的字符串。该类型可以是基本类型，如int、指针、标记结构或联合体，或类名——事实上，任何类型都可以用作C sizeof()运算符的参数。
```c
char *buf1 = @encode(int **);
char *buf2 = @encode(struct key);
char *buf3 = @encode(Rectangle);
```
下表列出了类型代码。请注意，其中许多与您为归档或分发目的对对象进行编码时使用的代码重叠。然而，这里列出了一些代码，您在编写编码器时不能使用，在编写编码器时，您可能想要使用一些代码，这些代码不是由@encode()生成的。（有关用于归档或分发的编码对象的更多信息，请参阅基础框架参考中的NSCoder类规范。）

| code | meaning                                                    |
| ---- | ---------------------------------------------------------- |
| c    | char                                                       |
| i    | int                                                        |
| s    | short                                                      |
| l    | A long is treated as a 32-bit quantity on 64-bit programs. |
| q    | long long                                                  |
| C    | unsigned char                                              |
| I    | unsigned Int                                               |
| S    | unsigned short                                             |
| L    | unsigned long                                              |
| Q    | unsigned long long                                         |
| B    | A C++ `bool` or a C99 `_Bool`                              |
| d    | double                                                     |
| f    | float                                                      |
| v    | void                                                       |
| *    | A character string (`char *`)                              |
| @    | An object (whether statically typed or typed `id`)         |
| #    | A class object (`Class`)                                   |
| :    | A method selector (`SEL`)                                  |
| [_array type_] |An array|
|{_name=type..._}|A structure|
|(_name_=_type..._)|A union|
|`b`num|A bit field of _num_ bits|
|`^`type|A pointer to _type_|
|`?`|An unknown type (among other things, this code is used for function pointers)     |                                                            |

重要信息：Objective-C不支持long double类型。@encode(long double)返回d，这与double的编码相同。
数组的类型代码包含在方括号内；数组中的元素数量立即在打开的括号之后，在数组类型之前指定。例如，一个由12个指针组成的数组将被编码为：
[12^f] 
