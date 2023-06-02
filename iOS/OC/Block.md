块类似于匿名函数或闭包，在许多其他编程语言中也存在类似的概念。
# Block
以下是块的一些基本知识：
1. **块的定义**：块是由一对花括号 `{}` 包围的代码片段，可以包含一段可执行的代码。块的定义使用 `^` 符号，并可以带有参数列表和返回类型。例如：
```objc
^{
    // 代码块的内容
}

```
2. **块的类型**：块也是一种数据类型，与函数类似。它们可以具有参数和返回值类型。可以使用 `typedef` 来定义块的类型。例如：
```objc
typedef returnType (^BlockTypeName)(parameterTypes);

```
其中 `returnType` 是块的返回类型，`BlockTypeName` 是块的类型名称，`parameterTypes` 是块的参数类型。
3. **块的赋值和调用**：块可以赋值给变量，并且可以像函数一样进行调用。可以使用 `=` 运算符将块赋值给变量，然后使用该变量调用块。例如：
```objc
ReturnType (^blockName)(ParameterTypes) = ^ReturnType (Parameters) {
    // 块的内容
};
blockName(argumentValues); // 调用块

```
4. **块的捕获变量**：块可以捕获其定义范围内的变量，并在块内部访问这些变量。捕获的变量在块中形成了一个闭包，可以在块的[[生命周期]]内保持其状态。例如：
```objc
NSInteger outsideVariable = 10;
void (^block)(void) = ^{
    NSLog(@"Outside variable: %ld", (long)outsideVariable);
};
block(); // 输出：Outside variable: 10
```
在这个例子中，块捕获了外部的 `outsideVariable` 变量，并在块内部访问它。
默认情况下，为块所捕获的变量是不可以在块里面修改的，如果修改了outsideVariale的值，就会报错，声明变量的时候可以加上__block修饰符，这样就可以在块内修改了
```objc

```
5. **块作为参数**：块可以作为方法或函数的参数进行传递，从而实现回调和异步操作等功能。可以将块作为参数声明，并在调用方法或函数时传递块。例如：
```objc
- (void)performOperationWithCompletion:(void (^)(void))completionBlock {
    // 执行操作
    // 操作完成后调用块
    completionBlock();
}

// 调用方法，传递块作为参数
[self performOperationWithCompletion:^{
    NSLog(@"Operation completed!");
}];

```
在这个例子中，`performOperationWithCompletion:` 方法接受一个块作为参数，并在操作完成后调用该块。
```objc
#pragma mark **--修改为块所捕获的变量**

    NSArray *array = @[@0, @1, @2, @3, @4, @5];

    **__block** NSInteger count = 0;

    //块作为方法的参数

    [array enumerateObjectsUsingBlock:^(NSNumber *number, NSUInteger idx, **BOOL** *stop) {

            **if** ([number compare:@2] == NSOrderedAscending) {

                count++;

            }

        NSLog(@"%ld====%@",(**unsigned** **long**)idx, number);

    }];

    NSLog(@"%ld", (**long**)count);

    //内联块的用法，传给“numberateObjectsUsingBlock:"方法的块并未先赋给局部变量，而是直接在内联函数中调用了，如果块所捕获的变量类型是对象类型的话，那么就会自动保留它，系统在释放这个块的时候，也会将其一并释放。这就引出了一个与块有关的重要问题，块本身可以视为对象，在其他oc对象能响应的选择子中，很多块也可以响应，最重要的是，块本身也会像其他对象一样，有引用计数，为0时，块就回收了，同时也会释放块所捕获的变量，以便平衡捕获时所执行的保留操作

    //如果块定义在oc类的实例方法中，那么除了可以访问类的所有实例变量之外，还可以使用self变量，块总能修改实例变量，那么除了声明时无需添加__block。不过，如果通过读取或写入操作捕获了实例变量，那么也会自动把self给捕获了，因为实例变量是与self所指代的实例关联在一起的。

    // 需要注意的是(self也是一个对象，也会被保留），如果在块内部使用了实例变量，块会自动对self进行保留操作，以确保在块执行期间保持对象的有效性。但是，如果在块内部直接使用了self，并对其进行读取或写入操作，那么self也会被捕获，从而导致循环引用的问题。为了避免循环引用，可以在块内部使用__weak修饰符来避免对self进行保留操作。
```
# 块的内部结构
