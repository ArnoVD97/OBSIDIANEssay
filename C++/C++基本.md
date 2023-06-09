# C++变量作用域
代码块内部声明的变量称为***局部变量***
在函数参数的定义中声明的变量称为***形式参数***
在所有函数外部声明的变量称为***全局变量***
-   **局部作用域**：在函数内部声明的变量具有局部作用域，它们只能在函数内部访问。局部变量在函数每次被调用时被创建，在函数执行完后被销毁。
    
-   **全局作用域**：在所有函数和代码块之外声明的变量具有全局作用域，它们可以被程序中的任何函数访问。全局变量在程序开始时被创建，在程序结束时被销毁。
    
-   **块作用域**：在代码块内部声明的变量具有块作用域，它们只能在代码块内部访问。块作用域变量在代码块每次被执行时被创建，在代码块执行完后被销毁。
    
-   **类作用域**：在类内部声明的变量具有类作用域，它们可以被类的所有成员函数访问。类作用域变量的生命周期与类的生命周期相同。
    

**注意：** 
如果在内部作用域中声明的变量与外部作用域中的变量同名，则内部作用域中的变量将覆盖外部作用域中的变量。

## 类作用域

类作用域指的是在类内部声明的变量：

## 实例

```c++
#include <iostream>   
class MyClass {  
public:  
    static int class_var;  // 类作用域变量  
};  
  
int MyClass::class_var = 30;  
  
int main() {  
    std::cout << "类变量: " << MyClass::class_var << std::endl;  
    return 0;  
}  
```

以上实例中，MyClass 类中声明了一个名为 class_var 的类作用域变量。可以使用类名和作用域解析运算符 :: 来访问这个变量。在 main() 函数中访问 class_var 时输出的是 30。