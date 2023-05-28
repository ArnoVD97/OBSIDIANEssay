将代码转化为可执行程序，通常需要经过三个步骤：编译，连接，运行。不同的语言，这三个步骤的操作也有些不同
目前只看oc，动态语言，在编译阶段并不知道变量的具体类型，也不知道真正调用的哪个函数，只有在运行期才会检查数据类型，同时在运行时才会根据函数名查找具体的要调用的函数，这样在程序没运行的时候，我们不知道调用一个方法具体会发生什么
oc吧一些决定性的工作从编译链接阶段推迟到 `运行时阶段`，甚至可以在程序运行时候动态修改一个方法的实现，这也为大为流行的「热更新」提供了可能性。
RUNTIME实际是一个库，这个库使程序在运行的时候动态创建对象，检查对象，修改类和对象的方法
# 消息机制的基本原理
对象方法调用都是类似 `[receiver selector];` 的形式，其本质就是让对象在运行时发送消息的过程。

我们来看看方法调用 `[receiver selector];` 在「编译阶段」和「运行阶段」分别做了什么？
编译阶段[receiver selector] 方法被编译器转化为
objc_msgSend(receiver, selector)不带参数
objc_msgSend(receiver, selector, org1, org2, ...)带参数
运行时：消息接受者receiver寻找对应的selector。
通过对应的[[isa指针]]找到receiver的Class类
在Class类的method list方法列表中找到对应的selector
如果在Class中找到这个selector，就继续在它的superClass类中寻找
一旦在找到对应的selector方法，直接执行receiver对应的selector方法实现的[[IMP方法实现]]
若找不到对应的selector，消息被转发或者临时向receiver添加这个selector对应的实现方法，