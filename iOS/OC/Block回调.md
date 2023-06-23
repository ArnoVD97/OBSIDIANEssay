[[Block]]
先跟着我实现最简单的 Block 回调传参的使用，如果你能举一反三，基本上可以满足了 OC 中的开发需求。已经实现的同学可以跳到下一节。

首先解释一下我们例子要实现什么功能（其实是烂大街又最形象的例子）：

有两个视图控制器 A 和 B，现在点击 A 上的按钮跳转到视图 B ，并在 B 中的textfield 输入字符串，点击 B 中的跳转按钮跳转回 A ，并将之前输入的字符串

显示在 A 中的 label 上。也就是说 A 视图中需要回调 B 视图中的数据。
![[Pasted image 20230623170202.png]]
![[Pasted image 20230623170242.png]]![[Pasted image 20230623170251.png]]
block回调传值是一个从后面向前传递值的一个过程，这个即这个块应该定义在B页面内，重命名一个块，你会的到两个函数，一个是调用函数，一个是回调函数，我们的网络请求用的就是block回调传值在网络请求中的manager就使用的是回调函数充当B页面，而需要调用数据的页面使用的是调用函数，可以获得回调函数中的数据，
```objc
    BVC.callBackData(NSString  _Nonnull data#);
//回调函数（传值函数）用用于传入回调数据

    BVC.callBackData = ^(NSString  _Nonnull data)
    //调用函数

//BViewController.h
#import <UIKit/UIKit.h>

typedef void(^CallBackBlcok) (NSString *text);//1

@interface BViewController : UIViewController

@property (nonatomic,copy)CallBackBlcok callBackData;//2
@end
```
在这里，代码 1 用 typedef 定义了 `void(^) (NSString *text)`的别名为 `CallBackBlcok` 。这样我们就可以在代码 2 中，使用这个别名定义一个 Block 类型的变量 `callBackBlock`。

在定义了 `callBackBlock` 之后，我们可以在 B 中的点击事件中添加 `callBackBlock` 的传参操作：
```objc

//BViewController.m

- (IBAction)click:(id)sender {
 self.callBackBlock(_textField.text); //1
 [self.navigationController popToRootViewControllerAnimated:YES];
}
```
这样我们就可以在想要获取数据回调的地方，也就 A 的视图中调用 block：
```objc

// AViewController.m
- (IBAction)push:(id)sender {
 BViewController *bVC = [self.storyboard instantiateViewControllerWithIdentifier:@"BViewController"];

 bVC.callBackBlock = ^(NSString *text){ // 1

  NSLog(@"text is %@",text);

  self.label.text = text;

 };
 [self.navigationController pushViewController:bVC animated:YES];
}
```
代码 1 中，通过对回调将 B 中的数据传递到代码块中，并赋值给 A中的 label，实现了整个回调过程。

上例是通过将 block 直接赋值给 block 属性，也可以通过方法参数的方式传递 block 块。
## Block 的疑惑

到目前为止，一切看起来都很美好（如果你照着上面的例子做的话），功能正常， A 视图中也获取到数据了。但是某些人可能就要说了，你的代码有问题，你的思路有问题，你这是误人子弟。

是的，代码的确还有问题，第一个问题就是循环引用的问题，在 A 视图的block 代码块中：
```objc

bVC.callBackBlock = ^(NSString *text){
  NSLog(@"text is %@",text);  
  self.label.text = text;  
 };
```
代码 `self.label.text = text;` ，在 Block 中引用 self ，也就是 A ，而 A 创建并引用了 B ，而 B 引用 `callBackBlock`，此时就形成了一个循环引用，而编译器也不会报任何错误，我们需要非常小心这个问题（面试百分百问到我会乱说？）。此时我们通常的解决方法是使用弱引用来解除这个循环：
```objc

 __weak AViewController *weakSelf = self;
 bVC.callBackBlock = ^(NSString *text){ 
  NSLog(@"text is %@",text); 
//  self.label.text = text; 
  weakSelf.label.text = text;
 };
```
第二个问题是我自己对 Block 的理解不到位，我们都知道 Block 能截取自动变量，并且是不能在 Block 块中进行修改的（除非用__block修饰符），但是很明显 `weakSelf.label.text`的值被修改了，并且没有用`__block`修饰符， 这是为什么呢？因为 label 是个全局变量，而如果像如下的局部变量 a 是不能修改的，编译器也会报错：

![](http://res.dedeyun.com/imgfile/2212/1C220U3D2310-54G8.jpg)  
局部变量  

通过这个小例子发现的两个问题，也算是值得了。