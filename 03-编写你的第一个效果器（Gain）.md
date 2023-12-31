# 前言

在上一章中，我们已经学会了如何配置参数。在这一章中，我们将设计一个Gain效果器，通过**多种不同的实现方式**来介绍参数的获取和Real-Time效果器的DSP部分基本写作逻辑。

你可以在这里找到本章中的相关代码：

[TaroStudio/TaroGain at main · TaroPie1214/TaroStudio (github.com)](https://github.com/TaroPie1214/TaroStudio/tree/main/TaroGain)

# 准备工作

在Projucer中添加module：juce_dsp

![iShot2023-10-09 11.11.31](https://cdn.jsdelivr.net/gh/TaroPie0224/blogImage@main/img/202310091111021.jpg)

根据之前的教程创建好工程并配置好我们这次需要的参数——Gain：

```cpp
juce::AudioProcessorValueTreeState::ParameterLayout TaroGainAudioProcessor::createParameterLayout()
{
    APVTS::ParameterLayout layout;

    using namespace juce;

    layout.add(std::make_unique<AudioParameterFloat>("Gain",
                                                     "Gain(dB)",
                                                     NormalisableRange<float>(-10.f, 10.f, 0.1f, 1), 0.f));

    return layout;
}
```

自动界面生成：

```cpp
juce::AudioProcessorEditor* TaroGainAudioProcessor::createEditor()
{
    return new juce::GenericAudioProcessorEditor(*this);
}
```

# processBlock

到达Real-Time效果器最高城！processBlock！

让我们先看看它默认长什么样：

```cpp
void AudioProcessor::processBlock (juce::AudioBuffer<float>& buffer, juce::MidiBuffer& midiMessages)
{
    juce::ScopedNoDenormals noDenormals;
    auto totalNumInputChannels  = getTotalNumInputChannels();
    auto totalNumOutputChannels = getTotalNumOutputChannels();

    // In case we have more outputs than inputs, this code clears any output
    // channels that didn't contain input data, (because these aren't
    // guaranteed to be empty - they may contain garbage).
    // This is here to avoid people getting screaming feedback
    // when they first compile a plugin, but obviously you don't need to keep
    // this code if your algorithm always overwrites all the output channels.
    for (auto i = totalNumInputChannels; i < totalNumOutputChannels; ++i)
        buffer.clear (i, 0, buffer.getNumSamples());

    // This is the place where you'd normally do the guts of your plugin's
    // audio processing...
    // Make sure to reset the state if your inner loop is processing
    // the samples and the outer loop is handling the channels.
    // Alternatively, you can process the samples with the channels
    // interleaved by keeping the same state.
    for (int channel = 0; channel < totalNumInputChannels; ++channel)
    {
        auto* channelData = buffer.getWritePointer (channel);

        // ..do something to the data...
    }
}
```

---

首先是：

```cpp
juce::ScopedNoDenormals noDenormals;
```

它的意思其实是禁用当前作用域下的浮点数的非规格化处理，可能有点绕口，但是这能在部分情况下显著提升音频的处理速度，当然也会伴随着一些负面影响。

关于具体什么是浮点数的规格化和非规格化处理，以及它们分别有什么利弊，可以查看以下这位老师的博客，讲解的十分清晰：

http://cenalulu.github.io/linux/about-denormalized-float-number/

**当然，如果你并不在乎为什么需要这行代码，我也鼓励你直接跳过来节省时间。**

---

```cpp
auto totalNumInputChannels  = getTotalNumInputChannels();
auto totalNumOutputChannels = getTotalNumOutputChannels();
```

获取了输入通道的数量和输出通道的数量。

---

```cpp
for (auto i = totalNumInputChannels; i < totalNumOutputChannels; ++i)
    buffer.clear (i, 0, buffer.getNumSamples());
```

将多余的输出通道内容置零，以防止啸叫产生。

---

```cpp
for (int channel = 0; channel < totalNumInputChannels; ++channel)
{
    auto* channelData = buffer.getWritePointer (channel);

    // ..do something to the data...
}
```

遍历所有的输入通道，并获取指向该通道下buffer的头部的指针，同时也是最关键的部分。

# Gain

Gain（增益）效果器是什么东西应该不用再具体介绍了，我们可以通过Gain来控制音频的响度。

接下来将用四种不同的方式来实现这个效果。

## 方式一

我们先了解一下如何获取apvts里面的参数值：

```cpp
float currentGain = *apvts.getRawParameterValue("Gain");    //括号里的字符需为之前提到的Parameter ID
```

回看刚刚提到的循环——遍历所有的输入通道，并获取指向该通道下buffer的头部的指针。

现在我们在这个循环中再嵌套一层对buffer的循环，以遍历buffer中的所有sample：

```cpp
// 通过buffer.getNumSamples()获取buffer里sample的数量
for (int sample = 0; sample < buffer.getNumSamples(); ++sample)
{
    // ...
}
```

对于每个sample，我们将它乘上我们的gain：

```cpp
// 注意我们的Gain定义的时候单位是dB，所以这里还需要一个转换
channelData[sample] *= juce::Decibels::decibelsToGain(currentGain);
```

整体代码如下：

```cpp
void TaroGainAudioProcessor::processBlock (juce::AudioBuffer<float>& buffer, juce::MidiBuffer& midiMessages)
{
    juce::ScopedNoDenormals noDenormals;
    auto totalNumInputChannels  = getTotalNumInputChannels();
    auto totalNumOutputChannels = getTotalNumOutputChannels();

    for (auto i = totalNumInputChannels; i < totalNumOutputChannels; ++i)
        buffer.clear(i, 0, buffer.getNumSamples());

    float currentGain = *apvts.getRawParameterValue("Gain");

    for (int channel = 0; channel < totalNumInputChannels; ++channel)
    {
        auto* channelData = buffer.getWritePointer (channel);

        for (int sample = 0; sample < buffer.getNumSamples(); ++sample)
        {
            channelData[sample] *= juce::Decibels::decibelsToGain(currentGain);
        }
    }
}
```

## 方式二

在这之前我们需要先了解一下JUCE的AudioBlock：

[JUCE: dsp::AudioBlock< SampleType > Class Template Reference](https://docs.juce.com/master/classdsp_1_1AudioBlock.html)

简单来说，它并不包含具体的数据，只是一个引用，我们通常用AudioBuffer来创建一个AudioBlock。在JUCE的dsp module中，block是非常关键的一部分，在之后的几个章节中，我们会经常使用它。

同理，在创建完block后，参考方式一，对block中的sample进行遍历并乘上gain。

整体代码如下：

```cpp
void TaroGainAudioProcessor::processBlock (juce::AudioBuffer<float>& buffer, juce::MidiBuffer& midiMessages)
{
    juce::ScopedNoDenormals noDenormals;
    auto totalNumInputChannels  = getTotalNumInputChannels();
    auto totalNumOutputChannels = getTotalNumOutputChannels();

    for (auto i = totalNumInputChannels; i < totalNumOutputChannels; ++i)
        buffer.clear(i, 0, buffer.getNumSamples());

    float currentGain = *apvts.getRawParameterValue("Gain");

    juce::dsp::AudioBlock<float> block(buffer);
    for (int channel = 0; channel < block.getNumChannels(); ++channel)
    {
        auto* channelData = block.getChannelPointer(channel);
        for (int sample = 0; sample < block.getNumSamples(); ++sample)
        {
            channelData[sample] *= juce::Decibels::decibelsToGain(currentGain);
        }
    }
}
```

## 方式三

其实，JUCE的AudioBuffer还自带一个方法叫做applyGain：

![未命名1696989921.png](https://cdn.jsdelivr.net/gh/TaroPie1214/blogImage@master/img/%E6%9C%AA%E5%91%BD%E5%90%8D1696989921.png)

使用该方法，我们可以直接对整个buffer进行gain的相乘。

整体代码如下：

```cpp
void TaroGainAudioProcessor::processBlock (juce::AudioBuffer<float>& buffer, juce::MidiBuffer& midiMessages)
{
    juce::ScopedNoDenormals noDenormals;
    auto totalNumInputChannels  = getTotalNumInputChannels();
    auto totalNumOutputChannels = getTotalNumOutputChannels();

    for (auto i = totalNumInputChannels; i < totalNumOutputChannels; ++i)
        buffer.clear(i, 0, buffer.getNumSamples());

    float currentGain = *apvts.getRawParameterValue("Gain");

    buffer.applyGain(juce::Decibels::decibelsToGain(currentGain));
}
```

备注：AudioBuffer作为JUCE的核心内容之一，JUCE为其提供了很多方法，这些方法可以在很多时候方便我们对buffer的处理，所以推荐对AudioBuffer的所有方法进行一个完整的预览，留个印象。

https://docs.juce.com/master/classAudioBuffer.html

## 方式四

这个方法涉及到JUCE的DSP模块使用，在此也通过Gain来引入其基本的使用方式。

首先在PluginProcessor.h的private部分创建一个Gain：

```cpp
// PluginProcessor.h
class TaroGainAudioProcessor  : public juce::AudioProcessor
{
public:
    // ...

private:
    // 方式4
    juce::dsp::Gain<float> gain;
    //==============================================================================
    JUCE_DECLARE_NON_COPYABLE_WITH_LEAK_DETECTOR (TaroGainAudioProcessor)
};
```

回到PluginProcessor.cpp，找到prepareToPlay。

在这里简单介绍一下prepareToPlay，它是插件在被初始化时执行的函数，当我们使用dsp modules时，我们会在其中告诉ProcessSpec当前的buffer大小，声道数量，采样率等等，并进行prepare和reset。

对于Gain来说，prepareToPlay的写法如下：

```cpp
// PluginProcessor.cpp
void TaroGainAudioProcessor::prepareToPlay (double sampleRate, int samplesPerBlock)
{
    // 方式4
    // 创建spec
    juce::dsp::ProcessSpec spec;
    // 告知spec当前的buffer大小，声道数量，采样率
    spec.maximumBlockSize = samplesPerBlock;
    spec.numChannels = getTotalNumInputChannels();
    spec.sampleRate = sampleRate;
    // 以这个spec中的信息来执行gain的处理
    gain.prepare(spec);
    gain.reset();
}
```

最后我们再来看processBlock。

同理，还是首先根据AudioBuffer创建一个AudioBlock，然后我们使用gain的setGainDecibels方法来告知它当前的增益大小，再通过process方法来执行gain效果。

注意，在将block送入process方法时，我们还对其进行了一个ProcessContextReplacing(block)的嵌套，不要忘记这个操作。

![未命名1696992985.png](https://cdn.jsdelivr.net/gh/TaroPie1214/blogImage@master/img/%E6%9C%AA%E5%91%BD%E5%90%8D1696992985.png)

整体代码如下：

```cpp
// PluginProcessor.cpp
void TaroGainAudioProcessor::processBlock (juce::AudioBuffer<float>& buffer, juce::MidiBuffer& midiMessages)
{
    juce::ScopedNoDenormals noDenormals;
    auto totalNumInputChannels  = getTotalNumInputChannels();
    auto totalNumOutputChannels = getTotalNumOutputChannels();

    for (auto i = totalNumInputChannels; i < totalNumOutputChannels; ++i)
        buffer.clear(i, 0, buffer.getNumSamples());

    float currentGain = *apvts.getRawParameterValue("Gain");

    juce::dsp::AudioBlock<float> block(buffer);
    gain.setGainDecibels(currentGain);
    gain.process(juce::dsp::ProcessContextReplacing<float>(block));
}
```

# 总结

你可以在这里找到本章中的相关代码：

[TaroStudio/TaroGain at main · TaroPie1214/TaroStudio (github.com)](https://github.com/TaroPie1214/TaroStudio/tree/main/TaroGain)

在这一章中，我们学习了Real-Time效果器的基本写法，比如AudioBuffer，AudioBlock的相关用法等。以及DSP module的使用方式（创建->prepareToPlay中配置spec，processBlock中进行process）等等。

接下来我们会继续深入各种常见效果器背后的DSP原理，并将其编写为JUCE效果器~
