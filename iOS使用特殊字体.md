# 为什么要使用特殊字体
我们的项目有时需要美工需求，系统自带的字体无法满足我们的需要，当我们的项目需要和美工对接的时候，这时候就得为项目添加我们的字体了
# 如何为项目添加字体
https://fonts.google.com
这个网站可以获取一些字体供我们使用
## 添加流程
### 第一步
我的建议添加到资源文件里面，为文件做好分类
比如我要添加***得意黑***字体
将这个文件添加进项目中![[Pasted image 20230410214015.png]]
ok
### 第二步
接下来要在项目中的`Info.plist`文件中添加新一行 AddRow 
Fonts provided by application然后添加key为item0，value为你刚才加入的SmileySans-Oblique.tff
### 最后
打开![[Pasted image 20230410214028.png]]
项目属性设置选项卡，build phases 将字体文件添加到copy bundle resources
以便在info文件中可以找到该文件的路径，其次就是确认一下字体文件的名称和font名称是否一致
