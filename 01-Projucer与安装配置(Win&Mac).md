# 获取JUCE

有两种方式可以获取JUCE（Cmake/Projucer），初学的朋友更推荐使用官方的构建工具Projucer

Projucer: [Download - JUCE](https://juce.com/download/)

Git: [juce-framework/JUCE: JUCE is an open-source cross-platform C++ application framework for desktop and mobile applications, including VST, VST3, AU, AUv3, RTAS and AAX audio plug-ins. (github.com)](https://github.com/juce-framework/JUCE)

# Projucer

## Global Paths配置

下载完成后会得到一个JUCE文件夹，对于Windows用户，如果你不嫌麻烦在之后配置Global Paths，且你并没有多年的开发经验的话，**我强烈建议你直接将这整个文件夹放在C盘的根目录下**，这是由于Projucer的默认设置就是指向这个位置的

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/2023/04/03/20230403-100919.png)

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/2023/04/03/20230403-100548.png)

Mac对应的默认位置：

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/2023/04/03/20230403-101136.png)

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/2023/04/03/20230403-101129.png)

如果你不想放在默认位置，请修改Path to JUCE和JUCE Modules指向你的自定义路径

## 创建第一个JUCE工程

点击左上角的FIle选择New Project...

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/2023/04/03/20230403-102305.png)

由于本系列教程主要以音频效果器插件为例，所以选择Projucer Plugin-In中的Basic模板

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/2023/04/03/20230403-102421.png)

输入你的项目名称，Modules暂时不用动，Path to Modules在上文我们已经配置好了，Exporters根据系统选择（Win->VS，Mac->Xcode），File Creation Options默认即可

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/2023/04/03/20230403-102737.png)

点击Create Project，选择生成路径后即可完成项目的创建，此时点击右上方的IDE图标即可打开对应IDE

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/2023/04/03/20230403-103143.png)

# 编译与运行

## Windows

点击上方的调试器开始编译

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/2023/04/03/20230403-103652.png)

会默认编译并打开standalone版本，如果一切顺利，你会看到这个界面

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/2023/04/03/20230403-104438.png)

停止调试，右键解决方案资源管理器中的VST解决方案并生成

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/2023/04/03/20230403-104638.png)

如果编译成功，我们可以在下图路径中找到结果

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/2023/04/03/20230403-104738.png)

将其复制到 *C:\Program Files\Common Files\VST3* 下，这是一般DAW的VST3默认扫描位置

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/2023/04/03/20230403-105252.png)

然后就可以在DAW的调音台中打开我们刚刚编译的插件了

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/2023/04/03/20230403-105333.png)

## Mac

在Xcode中，我们也先尝试编译Standalone版本

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/2023/04/03/20230403-105846.jpg)

会得到跟Windows端相同的内容

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/2023/04/03/20230403-110055.png)

更换解决方案，这里我们以AU为例

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/2023/04/03/20230403-110208.png)

编译完成后，跟Windows不同的是，在项目的build目录下生成的是替身，实际结果会直接build到系统统一放置**用户插件**的路径，如下

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/2023/04/03/20230403-110934.png)

但此时Logic会有概率扫描不到该插件，如果发生这种情况，请复制到**系统插件**路径下

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/2023/04/03/20230403-111255.png)DAW扫描到插件后，也就可以正常挂载了

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/2023/04/03/20230403-110919.png)

# 总结

在本章中，我们学习了在Win和Mac下使用Projucer并完成工程的基础配置，成功编译运行。

在下一章中，我们将学习如何将我们想要控制的参数添加到插件中，并自动生成对应的交互界面。
