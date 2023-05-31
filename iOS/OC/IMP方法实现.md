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
``