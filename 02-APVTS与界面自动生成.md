# 参数管理

在我们设计各类插件时，不可避免的会面对众多参数的管理问题（比如下图中一个简单的compressor就包含了Threshold，Attack，Release，Ratio等参数），对此JUCE提供了一种方法叫做AudioProcessorValueTreeState（以下简称APVTS）: 

https://docs.juce.com/master/classAudioProcessorValueTreeState.html

![image-20230925113623287](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/202309251136375.png)

# APVTS

在进入这部分内容的教程之前，请确保你已经完成了上一章的内容，即配置好了开发环境并创建了一个plugin工程。

打开PluginProcessor.h，在public部分补充以下内容：

```cpp
// PluginProcessor.h
public:
    // ...
    using APVTS = juce::AudioProcessorValueTreeState;
    static APVTS::ParameterLayout createParameterLayout();

    APVTS apvts {*this, nullptr, "Parameters", createParameterLayout() };
```

由于AudioProcessorValueTreeState这一长串字母实在太长，我们通常用using简化成APVTS

打开PluginProcessor.cpp，实现刚刚创建的createParameterLayout()方法，这里以一个compressor的相关实现为例：

```cpp
juce::AudioProcessorValueTreeState::ParameterLayout TaroCompressorAudioProcessor::createParameterLayout()
{
    APVTS::ParameterLayout layout;

    using namespace juce;

    layout.add(std::make_unique<AudioParameterFloat>(ParameterID { "Threshold", 1 },
                                                     "Threshold",
                                                     NormalisableRange<float>(-60, 12, 1, 1), 0));

    auto attackReleaseRange = NormalisableRange<float>(0, 500, 1, 1);

    layout.add(std::make_unique<AudioParameterFloat>(ParameterID { "Attack", 1 }, "Attack", attackReleaseRange, 50));

    layout.add(std::make_unique<AudioParameterFloat>(ParameterID { "Release", 1 }, "Release", attackReleaseRange, 250));

    auto choices = std::vector<double>{ 1.5, 2, 3, 4, 5, 6, 7, 8, 10, 15, 20, 50, 100 };
    juce::StringArray sa;
    for ( auto choice : choices )
    {
        sa.add( juce::String(choice) );
    }

    layout.add(std::make_unique<AudioParameterChoice>(ParameterID { "Ratio", 1 }, "Ratio", sa, 3));

    return layout;
}
```

接下来我们对以上代码进行拆解

假设我们有一个参数叫做Threshold，它的最小值为-60dB，最大值为12dB，最小步进（参数可调整的最大精度）为1dB，默认值为0dB。

```cpp
layout.add(std::make_unique<AudioParameterFloat>(ParameterID { "Threshold", 1 },
                                                 "Threshold",
                                                 NormalisableRange<float>(-60, 12, 1, 1), 0));
```

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/202309251604164.png)

我们首先来看一下AudioParameterFloat，在这里它接收了四个参数，分别是：

- parameterID：参数的ID，必须唯一，我们通过这个ID在别处去修改或获取参数的值

- parameterName：参数的名称，用于在交互界面上进行展示等等

- normalisableRange：参数的范围，它也接受了四个参数，分别是
  
  1. rangeStart：最小值
  
  2. rangeEnd：最大值
  
  3. intervalValue：步进
  
  4. skewFactor：斜率因子
     
     - 大部分情况下都是1，如果参数绑定了一个滑块或者旋钮，那么1就表示随着滑块或旋钮的移动，参数均匀变化
     
     - 小部分情况下需要手动调整，比如该参数是EQ效果器中被衰减或增益的一个频点，那么在较低频率时参数应该变化较慢，在较高频率时，则应该变化较快，这一点在大部分EQ效果器上的横轴分布就有体现
       
       ![image-20230925161504441](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/202309251615581.png)

- defaultValue：默认值

在这里你会发现，在parameterID的位置，我并没有直接给出一个字符串，而是使用了

```cpp
ParameterID { "Threshold", 1 }
```

请注意，这样的写法只在编译AU（Audio Unit）格式的插件时需要被使用，其它情况，如Standalone，VST3等则不需要这个步骤，直接给出字符串即可（当然你全都这么写上也不冲突）

```
layout.add(std::make_unique<AudioParameterFloat>("Threshold",
                                                 "Threshold",
                                                 NormalisableRange<float>(-60, 12, 1, 1), 0));
```

我们再来看一个AudioParameterChoice，它跟AudioParameterFloat是最常用的两种APVTS参数类型

![image-20230925164447235](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/202309251644335.png)

它也接收了四个参数，分别是：

- parameterID
- parameterName
- choices：一个StringArray字符串数组，包含所有的选项
- defaultItemIndex：默认选项的Index

```cpp
auto choices = std::vector<double>{ 1.5, 2, 3, 4, 5, 6, 7, 8, 10, 15, 20, 50, 100 };
juce::StringArray sa;
for ( auto choice : choices )
{
    sa.add( juce::String(choice) );
}
layout.add(std::make_unique<AudioParameterChoice>(ParameterID { "Ratio", 1 }, "Ratio", sa, 3));
```

在这里，我们首先创建了一个juce::StringArray数组，并把所有的Ratio可选值添加到了其中，然后再以此创建了一个AudioParameterChoice并添加到了layout，其中默认选项的index为3，所以在插件初始化时，Ratio的值会默认是4（0->1.5，1->2，2->3，3->4，...）

# 自动界面生成

当我们创建完APVTS之后，此时对代码进行编译，显示的依然还是默认的Hello World，因为我们并没有对PluginEditor进行任何修改。

在之后，我们会讲到如何创建自定义控件并与我们在APVTS中创建的参数进行绑定，并自由设计和摆放我们的控件。但创建控件->绑定参数->摆放控件这一过程还是略显繁琐，如果我们只是在插件的测试阶段（只需要测试它的dsp效果等），其实完全可以跳过这个步骤。

![](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/202309251700294.png)

在这里，JUCE提供可一种自动构建界面的方法。

打开PluginProcessor.cpp，找到以下内容：

```cpp
juce::AudioProcessorEditor* TaroCompressorAudioProcessor::createEditor()
{
    return new TaroCompressorAudioProcessorEditor (*this);
}
```

把它替换成：

```cpp
juce::AudioProcessorEditor* TaroCompressorAudioProcessor::createEditor()
{
    return new juce::GenericAudioProcessorEditor (*this);
}
```

由此JUCE会根据你当前在APVTS中创建的所有参数自动构建界面，自动生成对应的Slider或ComboBox，如果你按照上述的代码构建了APVTS，那么你此时应该会看到这样的界面：

![image-20230925113623287](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/202309251705638.png)

# 总结

你可以在这里找到本章中的相关代码：

[TaroPie1214/TaroStudio: A collection of all my plugins! (github.com)](https://github.com/TaroPie1214/TaroStudio/tree/main/TaroCompressor)

在本章中，我们首先学习了JUCE中常用的参数管理方式——APVTS，了解了如何向其中添加我们想要的参数，然后根据这些参数自动生成对应的界面。

在下一章中，我们将开始窥探dsp部分的内容，由此创建我们第一个效果器！
