---
layout: post
title: 音频相似度计算初探
datetime: 2019-03-28 22:21:32
description: 基于python计算音频相似度
comments: true
tags:
 - Python
categories:
 - Python
---



> 这个是很早以前做的一个需求：**比较两个音频在频谱上的相似度**。最近整理笔记本的时候，翻了出来，顺便整理了一下当时的需求设计思路。当时查阅了一些相关的文章，在音频识别中，最基础的几个点是：时域、频域、短时能量和短时过零率等。google了一些网上的讨论，发现大部分人比较赞同通过比较音频的“短时能量”来判断i两个音频的相似程度。

对于一个音频而言，得到音频的短时能量分布，流程大致如下：

1. 解码音频到wav

2. 获取音频参数和数据

3. 归一化处理原始数据

4. 对音频数据做高通滤波，过滤掉低频信号的干扰

5. 求得该音频的短时能量分布

   

## 获取音频数据

获取wav音频数据，每16位读取一次带符号数作为一次采样点的采样结果。

![](/images/posts/compare_wav/raw_spectrum.png)



## 归一化采样数据

获取所有采样点最大数值max_value（绝对值最大值），通过`max_value`对所有采样点数据做归一化处理。因为音频各个采样点对应的幅值分布会比较广，所以需要通过归一化处理，将信号转换为同意的标准模式，将所有采样点的数据幅值调整到[-1,1]之间，归一化过程如下：


$$
x_t(i)=\frac{x(i)}{|max(x(i))|}
$$




>  其中，x(i)为第i个采样点原始数据的幅值 (-32768, 32767)。$$x_t(i)$$ 为第i个采样点数据归一化后的幅值(-1, 1)
>



归一化后的结果如下:

![](/images/posts/compare_wav/Raw_Spectrum(normalization).png)



## 过滤低频干扰

设计高通滤波器过滤低频干扰。由于语音信号的平均功率谱受声门激励和口鼻辐射的影响，语音信号从嘴唇辐射后，高频端大约在800Hz以上有6dB/倍频的衰减，因此，在对于语音信号进行分析之前，一般要对语音信号加以提升（预加重）。预加重的目的是为了提升高频部分，弱化低频，使信号频谱变得平坦，以便后续进行频谱分析和声道参数的分析。



在音频处理中，存在一种一阶的高通滤波器，以预加重的方式滤除掉50Hz的低频干扰。这里运用一个6dB/倍频的一阶高通滤波器：


$$
y(n)=1.0*x(n)-u*x(n-1)
$$


其中x(n)为原始音频数据序列，y(n)为通过滤波器预加重后的数据序列，u为预加重系数。它的范围可取在0.9-1.0之间。这里根据音频处理方向公开的经验取 u=0.9375。
滤波后的效果如下：

![](/images/posts/compare_wav/Wave_Filtering.png)



python代码如下：

``` python
def get_wave_filtering(self):
    """
        求滤波
        y(n) = 1.0*x(n)+(-0.9375)*x(n-1)

    :return:
    """
    #TODO 公式原理待验证，优化
    datause_n_2_list = [0.00] + list(self.datause[:-1])
    datause_n_2 = np.array(datause_n_2_list)

    self.datause = self.datause * 1.0 + (-0.9375) * datause_n_2
    self.get_normalization_data()

    #使用signal做高通滤波
    # b, a = signal.butter(1, 0.9375, 'low')
    # self.datause = signal.filtfilt(b, a, self.datause)
```



## 创建hamming窗口

至于为什么要使用hamming窗，是因为：

1. 音频数据很长，不可能一次性处理完，所以需要分段，一段一段的处理
2. 分段后的音频没有明显的周期性，不方便做后续求卷积，但是加上hamming窗口，数据形状就有了明显的周期性。一个窗内数据就代表一个周期

这里引用一个典型的hamming窗函数：


$$
h(n,a)=(1-a)-a*\cos(2*PI*\frac{n}{N-1}), 0{\leq}n{\leq}N-1
$$




一般情况下，选取a=0.46，效果图如下：

![](/images/posts/compare_wav/Wave_Filtering2.png)

加上hamming窗之后，窗内中间的数据会体现出来，两侧的数据信息会丢失，所以在做卷积时，每次只会移动1/3或者1/2个窗，这样被前一帧丢失的数据又重新得到了体现。

python代码实现如下

``` python
@staticmethod
def get_generate_hamming_windows(row, column):
    """
        创建hammin窗口

    :param row:
    :param column:
    :return:
    """

    hammin_windows = [0] * row * column
    for index in range(row * column):
        hammin_windows[index] = 0.54 - 0.46 * (math.cos(2 * math.pi * index / (row * column - 1)))
    return np.array(hammin_windows)

    # 也可直接使用numpy的hamming方法直接获取窗口函数
    # hammin_windows = np.hamming(32 * 16)
```



## 获取短时能量分布

语音信号和噪音信号的主要区别在于他们的能量。语音段的能量比噪音段的大，语音段的能量是噪音段能量和语音声波能量的和，所以在噪音比较小的情况下，计算输入信号的短时能量就能通过设置阈值把语音段和背景噪音段区分开。短时能量可以看做是语音信号的平均经过了一个线性滤波器的输出。

短时能量可以有效地判断信号幅度的大小，并可以用于进行有声/无声的判断。语音信号的能量随着时间变化会比较明显，其短时能量分析给出了反应这些幅度变化的一个合适的描述方式。这就为我们做音频相似度对比提供了很好的依据。
对于信号x(n)，短时能量的定义如下：
【公式】

### 去除首位静音片段，获取有效能量区间

如上所示，音频的短时能量分布可以用于进行有声/无声的判断。如上图所示，妲己的原声，在末尾有很长一段静音片段，这一个片段对于我们不仅没有意义，还会干扰我们的判断，所以在做相似度判断之前，需要先去掉这部分数据	

![](/images/posts/compare_wav/Wave_Filtering1-1.png)

这里，选择能量值低于0.025的采样点，认为是静音。然后通过这个阈值对音频首位进行扫描，去除首尾的静音片段，得到妲己原声的真正的有效的短时能量分布：



![](/images/posts/compare_wav/Useful_Short_Time_Energy.png)



###对比音频短时能量，获取相似度

利用上述方法，可以获取到原始音频和用户音频的有效短时能量分布。那么如何对比两个数据矩阵的相似度呢？这里想到了计算文本相似度中常用的方法：余弦相似度。

余弦相似度常用于文本挖掘中的文件比较，比如比较两段文字的相似度，论文的相似度等。这里就不赘述了。



>  通过上述方法，获取到该用户录音的有效短时能量。然后计算余弦距离,得到音频相似度。



对于用户录音来讲，可能有效片段会比原音的有效片段要短，那这时需要在末尾加0，保持原音和录音的短时能量的采样点数一致，方便后续求余弦相似度

余弦相似度公式如下：


$$
similarity=\cos(\theta)=\frac{A*B}{||A||||B||}=\frac{\sum_{i=1}^n{A_i{\times}B_i}}{\sqrt{\sum_{i=1}^n{(A_i)^2}{\times}\sum_{i=1}^n{(B_i)^2}}}
$$


