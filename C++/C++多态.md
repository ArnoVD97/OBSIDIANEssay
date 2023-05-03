**多态**按字面的意思就是多种形态。当类之间存在层次结构，并且类之间是通过继承关联时，就会用到多态。

C++ 多态意味着调用成员函数时，会根据调用函数的对象的类型来执行不同的函数。

下面的实例中，基类 Shape 被派生为两个类，如下所示：
```c++
#include <iostream>

**using** **namespace** std;

**class** Shape {

   **protected**:

      **int** width, height;

   **public**:

      Shape( **int** a=0, **int** b=0)

      {

         width = a;

         height = b;

      }

      **int** area()

      {

         cout << "Parent class area :" <<endl;

         **return** 0;

      }

};

**class** Rectangle: **public** Shape{

   **public**:

      Rectangle( **int** a=0, **int** b=0):Shape(a, b) { }

      **int** area ()

      {

         cout << "Rectangle class area :" <<endl;

         **return** (width * height);

      }

};

**class** Triangle: **public** Shape{

   **public**:

      Triangle( **int** a=0, **int** b=0):Shape(a, b) { }

      **int** area ()

      {

         cout << "Triangle class area :" <<endl;

         **return** (width * height / 2);

      }

};

// 程序的主函数

**int** main( )

{

   Shape *shape;

   Rectangle rec(10,7);

   Triangle  tri(10,5);

   // 存储矩形的地址

   shape = &rec;

   // 调用矩形的求面积函数 area

   shape->area();

   // 存储三角形的地址

   shape = &tri;

   // 调用三角形的求面积函数 area

   shape->area();

   **return** 0;

}
```