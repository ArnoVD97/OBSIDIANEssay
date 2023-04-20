使用事例：M1芯片，macOS Ventura 13.0.1
# Unity下载安装
`Unity`的安装直接在官网（[https://unity.cn/](https://unity.cn/)）下载`Unity Hub`，然后使用`Unity Hub`安装对应版本的`Unity`即可
# VSCode配置
代码编辑器我喜欢用`VSCode`，它比`Visual Studio`轻量美观，而且插件丰富，推荐。  
`VSCode`官网：[https://code.visualstudio.com/](https://code.visualstudio.com/)
直接下载VSCode
https://`vscode.cdn.azure.cn`/stable/da15b6fd3ef856477bf6f4fb29ba1b7af717770d/VSCode-darwin-universal.zip
## VSCode插件
### unity3d-pack插件
unity3d-pack插件是一个插件套件，它会自动安装对应的依赖插件：C# Snippets、C# XML Documentation Comments、Unity Code Snippets、Shader language support for VS Code、Unity Tools、Debugger for Unity、ShaderlabVSCode

##### Bracket Pair Colorizer 2插件

这是一个括号颜色匹配的插件，方便代码的阅读，也建议大家安装。
##### 其他插件

如果你的项目有使用`lua`，你还需要安装一下`lua`相关的插件，我一般安装的是：  
`EmmyLua`和`Lua`这连个插件
然后为了方便查看工程目录结构，会再安装一个`vscode-icons`插件，
## Mac配置Mono
`mono`下载：[https://www.mono-project.com/download/stable/](https://www.mono-project.com/download/stable/)
下载安装完毕后，设置`.zshrc/.bash_profile`环境变量(很重要)
```java

export FrameworkPathOverride=/Library/Frameworks/Mono.framework/Versions/Current

```
在终端执行`mono`，如果输出参数提示，则说明`mono`安装成功了。
## 安装.NET Core SDK
你的`VSCode`估计会弹出这个提示：`.Net Core SDK`找不到。
使用brew来安装.net sdk
`brew install dotnet-sdk`
执行dotnet --version来看dotnet版本
## 设置External Script Editor为VSCode

最后记得把`Unity`的`External Script Editor`设置为`Visual Studio Code`，如下
![[Pasted image 20230420202639.png]]
## 安装JRE
有些程序可能需要依赖`Java`运行环境，需要安装`JRE`，  
官网下载：[https://www.java.com/zh-CN/download/](https://www.java.com/zh-CN/download/)
## 安装HomeBrew
上面安装dotnet不成功（没有brew指令）
可以直接通过`HomeBrew`官网的命令行来安装，  
地址：[https://brew.sh/](https://brew.sh/)
不出意外的话，你遇到`443`错误，可以通过`gitee`镜像源来安装：
```
/bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/Homebrew.sh)"

```

![[Pasted image 20230420202844.png]]

安装完成后执行brew -v
输出版本号则说明安装成功了
接着我们就可以使用`brew`愉快地安装软件了，比如
brew install git
```
# 替换成阿里巴巴的 brew.git 仓库地址:
cd "$(brew --repo)"
git remote set-url origin https://mirrors.aliyun.com/homebrew/brew.git

# 还原为官方提供的 brew.git 仓库地址
cd "$(brew --repo)"
git remote set-url origin https://github.com/Homebrew/brew.git

# 替换成阿里巴巴的 homebrew-core.git 仓库地址:
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://mirrors.aliyun.com/homebrew/homebrew-core.git

# 还原为官方提供的 homebrew-core.git 仓库地址
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://github.com/Homebrew/homebrew-core.git

```
## 安装oh-my-zsh
上面提到`zsh`，我用的是`oh-my-zsh`，相信大家用了`oh-my-zsh`也会爱上它。