# 一、**DSP Filter基础**

## （1）基础知识

我们的TX，本质上，都是希望通过天线，传递一系列Bit的数字信号的。这样就必须考虑TX传递的准确性。譬如，有一个数字10需要传递，则我们应该传递的是10的二进制信号1010。需要先将这个二进制信号1010传入到我的TX内，我的TX再通过**DAC-Low Pass Filter- Mixer - PA**这样的路径去实现发射。在DTX中，信号1010可以直接DTX内实现上述功能。

众所周知，DTX本质上是一个DAC，即，将一个1010序列的数字信号量化为10模拟单位的模拟信号 **（DAC）**，并且功率放大 **（PA）** ，以RF的频率 **(Up-Mixer)** 在Antenna输出。

现在需要考虑的是，1010序列的数字信号怎么给到DTX中？

假设我们设计并且流片得到了一个DTX芯片，那么通过**FPGA将这4Bit信号通过接口Pad打入芯片中**是最合适的。这种信号显然是并行或者串行的 **方波** 。也就是0阶保持ZOH得到的波形。

对于**单个方波**，其傅里叶变换为：
$$

\mathscr{F}[x(t)]=h\tau Sa (\frac{\tau\omega}{2}) \tag{1-1}

$$
其中$h$是方波的幅度，$\tau$是方波的width。

**这里需要改变一下思维**：不论是串行1010的一个方波，还是并行的1010四个方波，它们都只是信号的一种形式，我们的信号本身是1010。更本质地，我们可以用impulse冲激信号表示它们。
若是串行serial：
$$

x[k] = 1\times \delta[k-3] + 0\times \delta[k-2] + 1\times \delta[k-1] + 0\times \delta[k] \tag{1-2}

$$
更一般性地表示为：
$$

x[k] = \sum_{n=0}^{N-1}s[n]\delta[k-n] \tag{1-3}

$$
若是并行parallel，则分为N路：

$$

x_n[k]=\sum_{n=0}^{N-1}s[n]\delta[k] \tag{1-4}

$$
其中$s(n)$就是我们要传的数字序列1010，**它们才是事实上的信号。**

>注意，有时候可能需要根据设计需求调整 $s(n)$ 的顺序，可能是从低位（LSB）到高位（MSB）或者从MSB到LSB。即有时候 $s[n]= 1010$，这种最先读取LSB的叫做小端法（little endian）， 有时候 $s[n] = 0101$，这种最先读取MSB的叫做大端法（big endian）

## （2）抑制replicas和spurs

所以，本质上，从FPGA打进芯片的方波并不是Original Signal。Original Signal(原始信号)是式（1-2）至(1-4)提及的 $s(n)$。或者可以用提及的delta累加和序列表示。

而**FPGA通过 ZOH 将Original Signal输出的**，这样信号就由**原本的离散Delta序列**，变成了**连续时间域（Continuous-Time Domain）的方波**。你也可以将这个**ZOH也视作为一种DAC**，其输出$y(t)$可以写作：
$$

y(t) = \sum_{k} x[k]h_{ZOH}(t-kT_s)

$$
其中$T_s$是采样周期。而ZOH的Impulse Response是一个“保持一个周期”的矩形：
$$

h_{ZOH}(t)=1, only \ when \ 0 \le t < T_s

$$
其**时域上就是一个方波**，在**频域**上由式（1-1）可以快速得出其傅里叶变换：
$$

\mathscr{F}[h_{ZOH}(t)]=H_{ZOH}(\omega)= T_s Sa(\frac{T_s}{2}\omega)

$$
如果要更严谨，还要考虑相位：
$$

H_{ZOH}(j\omega)=\int_{0}^{T_s}e^{-j\omega t}h_{ZOH}(t)dt = \frac{1-e^{-j\omega \frac{T_s}{2}}}{j\omega} = T_s Sa(\frac{T_s}{2}\omega) e^{-j\omega \frac{T_s}{2}}

$$
- 幅度上：ZOH会在频率 $\omega$ 处，让信号产生大小为 $T_s Sa(\frac{T_s}{2}\omega)$ 的杂散。

- 相位上：ZOH会在频率 $\omega$ 处，给信号提供一个半周期time delay $e^{-j\omega \frac{T_s}{2}}$

因此，不论输入信号以多少的频率输入，**经过ZOH，频域上总会有一个sinc包络的杂散**（时域上**卷积**了方波，则在**频域**上会乘积sinc包络）。

那么这个sinc包络的杂散送进电路中会造成什么后果呢？这就不得不回顾一下DPA的工作机理了。

最简单的DPA是这样设计的：在Output Matching之前，所有电路模块的行为都是开关行为，也就是说，这之间传递的一直是方波。直到具有选频作用的Output Matching network输出匹配网络，将carrier frequency载波频率部分提取出来。如果这个Output Matching是**LC单频点滤波**的，我的方波自然可以送进来，sinc spurs（sinc包络杂散）自然被滤除；
![](assets/DSP%20Filter方案与设计记录/file-20260120214007158.png)
但是如果Output Matching的滤波mask不是单点的呢？Namely，如果Output Matching的滤波是一个宽带的滤波，可以让sinc spurs大部分都通过的话，输出岂不就带有很高的杂散了吗？
因此！！！我们需要对原本的方波进行**数字滤波**，再将滤波好的信号进行DAC处理。

**这里同样有一个很容易理解错误的点**，数字滤波，难道是我将方波滤波成一个别的波形吗？这种想法是错的。实际上，在低频的数字模块中，它只有0和1的跳变，理论上都应该是很理想很漂亮的方波，因此**在数字域中，用时域单个信道波形的角度去看DSP的结果是不合适的。**

实际上，数字滤波确实是会改变信号的波形，但不是“方波”$\rightarrow$“类方波”的改变。DSP中FIR滤波器的原理是卷积：
$$

y[k] =

\sum_{n=0}^{N-1}

x[k]h[n-k]

$$
其中k代表离散点的离散时刻；N是FIR的抽头系数，一定程度上代表了FIR滤波的锋利度。$x[k]$是输入，要注意它本身是1010四个的输入的十进制数，即4'd10。并且在单个时间点看它也是不利于理解的，应该在一个比较宽泛的时域观察，譬如，视作是要输入1,7,5,**10**,3,7,5这种输入给DAC的离散数字。**滤波会将这样的波形的包络更加圆滑，但是各自的数据点还是离散的点**。
![](assets/DSP%20Filter方案与设计记录/file-20260121110607665.png)
## （3）抑制码间干扰（ISI）

ISI是指，**在一个通道中，数字信号进行切换时，由于上一个周期会残有能量，同时数字脉冲（譬如方波）由于带宽受限，其无法保持理想的跳变，它的下降沿会延伸到下一个信号的周期，这些都会影响下一个周期的信号。**

形象的例子是，在学verilog的时候，会看到所有的方波的边缘都不是“竖直“着下来的，而是斜着下来的（`如下图所示`），这个斜着下来的部分进入下一个采样周期，该部分被称作**拖尾**。
![](assets/DSP%20Filter方案与设计记录/file-20260121110808230.png)
解决方案是，**在$f = f_s$处不进行采样**。核心是利用Nyquist 升余弦滤波器。

## (4) 插值的方式以及相关用法

数字基带中，可以用的插值方式主要有：插零插值（Insertion of Zero, **IOZ**），零阶保持（Zero Order Hold, **ZOH**)。
### i. IOZ插零

IOZ的对频谱的影响是，将原频谱在原sampling rate的整数倍处，展现采样镜像。可以理解为，插零本质上是让**离散时域更接近连续时域**，则**频谱更趋近于连续域频谱**，故而离散频谱在真实频谱上的**周期延拓会被显示出来**。

可以利用这个特性，在Matlab仿真的时候，通过插0的方式查看到真实频谱的周期延拓的部分。方式如下所示：
```matlab

sig_ioz = upfirdn(sig_org, 1, 5); % interpolation of L = 5

```
![](assets/DSP%20Filter方案与设计记录/file-20260121111217660.png)

### ii. ZOH

此处ZOH强调的是插零方式，与先前数字信号以方波（ZOH）的方式传输有不同的侧重点。**频谱上相当于和sinc包络相乘**。

如果原始信号是$x_0[n]$，经过ZOH插值后的信号是$x_1[n]$，原始信号的频率为$f_0$，经过插值因子$L$之后信号的频率变为$L\times f_0$。此处可以视作用一个$L \times f_0$ 频率的脉冲序列去采样一个经过ZOH后的原始信号$x(t)$。

可以简单知道，$x(t)$的频谱具有一个以$f_s$为零点的sinc包络，用$L \times f_0$采样，sinc包络的零点仍旧在原处，但是由于采样的周期延拓，会在$L \times f_0$处生成一个采样镜像，sinc包络具有无穷多个旁瓣，因此总会有aliasing。但是如果$L$够大，譬如$L=5$，则是第2个旁瓣和镜像的第2个旁瓣aliasing，主瓣和第4个旁瓣aliasing。因此$L$较小的时候，得到的sinc包络是不正确的，当$L$较大的时候，受到的影响可以忽略。
## Question & Solution

- **1. Q**：对于数字滤波，如果我input的是ZOH的信号，数学上频带有Sinc包络的杂散，因此需要用滤波器将杂散压下去；但是如果数字滤波器的输出又是方波（相当于ZOH）的话，岂不是又会给输出加一个Sinc包络的杂散吗？
  **1. S**：是的，是会加入一个Sinc包络的杂散。更加准确点说，是相乘。
回顾流程：
-- （1）. 假设一个单频信号，经过采样后，频谱上会有周期延拓，此时频谱为$H_1(f)$。
-- （2）. 经过ZOH，则相当于$h_1(t)$的信号与ZOH在时域上卷积，同时在Freq.域上乘积。得到了一个含有sinc包络的离散频谱$H_2(f)$。
-- （3）. 经过Filter，杂散部分被压得很低，得到得频谱是$H_3(f)$
-- （4）. 即使此时输出还是ZOH的方式，相当于$H_3(f)$再乘以一个Sinc函数，此时杂散频点的幅值已经被压低了，所以即使再乘一个Sinc函数，也不会让杂散高起来。
* * *
- **2. Q**：如果我的序列是一个随机序列，它的频谱是一个分布广泛的频谱，如下图所示：

![d3b31deeb52a863e2bffc5bf3f60c051.png](en-resource://database/571:1)

那么在经过ZOH以后，添加了一个sinc包络，如下图所示：

![b3ffabfd3cf5a666b0aad689977b95c2.png](en-resource://database/570:1)

可见高频部分被压制了。随后，还需要用LPF，压制Sinc高频部分，这样的话（1）我上述随机序列的频谱，岂不是无法复原了？（2）为什么要用LPF压抑sinc包络？

**2. S**:

（1）是的，随机序列的频谱是无法还原，而且不需要还原。需要还原的是离散信号序列，即$s[n]$，即使LPF会改变$s[n]$，但是RX端是可以利用RX端的技术，将这段带内可逆的线性失真**解卷积**。

（2）之所以需要用LPF压抑sinc包络，是因为，即使拥有一个无限宽带的TX，也不能将这种无线宽带的信号发射。这涉及到的是现实性的问题，为了避免不同用户之间的带内干扰，所有的发射协议都规定有一定的mask，所有的发射机均要满足其对应的mask，这已经是法律上的要求了。

  

  

# 二、Filter Selection

Filter可以分为多种类型，从数学实现角度，分为FIR滤波器和IIR滤波器两类。

  

其中FIR滤波器需要较高的滤波阶次，即所需要的抽头系数较高；IIR滤波器可以在较小的滤波阶次实现滤波效果，但是稳定性和线性相位特性较低。因此我们会选用FIR滤波器。

  

以滤波特性而言，滤波器包括很多种。在统一使用FIR滤波器的情况下，不同滤波特性的滤波器决定了滤波器的脉冲响应，即：

$$

y[k] = \sum_{n=0}^{N-1}{x[n]h[k-n]}

$$

其中的$h[n]$则由不同滤波器的种类决定。

以下是考虑使用的方案：

## 1. Nyquist滤波器

### （1）Intro

  

Nyquist Filter的作用是，能够尽可能减小”符号级无码间干扰（Inter-Symbol Interference，ISI）“。

### （2）Mechanism

Nyquist Filter 理想模型：Sinc脉冲

$$

h(t) = sinc(\frac{t}{T_s}) \tag{2-1}

$$

**频域上**，sinc函数和方波是FT pair，具有最sharp的滤波作用。

**时域上**，采样时刻t=nT时，只有n=0时不为0，其他采样时刻都为0，即使此时有symbol的展宽，此时也不会被采样。

**但是它难以被实现**，原因是sinc函数在时域上无限长。

* * *

#### a. **Raised Cosine Filter(升余弦滤波器)**

由于Sinc脉冲滤波器是impractical的，因此会给它一个变量系数去拟合，这个变量系数是Raised Cosine函数:

$$

f_{RC}(t)=\frac{\cos{(\pi\beta t / T_s)}}{1-(2\beta t / T_s)^2} \\

h(t) = sinc(\frac{t}{T_s})\times f_{RC}(t)

$$

其中$\beta$是滚降因子，滚降因子越小，$h(t)$越接近于$sinc(t/T_s)$。这里$h(t)$的傅里叶变换为：

$$

H(f)= \left\{\begin{array}

{l}

1, \ |f| \le (1-\beta)\frac{f_s}{2}\\

\frac{1}{2f_s}[1+\cos(\frac{\pi}{\beta f_s}(|f|-\frac{(1-\beta)f_s}{2}))], \ (1-\beta)\frac{f_s}{2} \le|f| \le (1+\beta)\frac{f_s}{2} \\

0,\ otherwise

\end{array}\right.

$$

滚降因子$\beta$越小，滤波器的频率响应中，dB=0的通带带宽也越大。

  

它的作用通常是将理想数字信号（即离散序列），变成一个ZOH输出的连续波形，该操作称作**脉冲成型**

  

  

  

## 2. 多相插值Filter的电路层实现方法

对于FIR插值滤波器，其Interpolation Factor = L，如果input freq. = $f_s$，则output freq. = $L\times f_s$，先从数学层面理解：

### (1) mathematic derivation

下列是相关参数

***

**input**: $x[k]/ X[z]$

**interpolation factor**: $L$

**FIR impulse response**: $h[n]/ H(z)$

**target output**: $y[z]/ Y(z)$

**插0后的input**: $x_1[k]/X_1[z]$

***

  

#### step i. 插0

假设这个插0+滤波是分成2 steps，first step就是插0：

$$

x_1[k] = \left \{ \begin{array} {l}

x[\frac{k}{L}], k = \pm L, \pm 2L, \pm 3L ...\\

  

0,\ otherwise

\end{array}\right.

$$

对上式进行z域变换：

$$

X_1[z] = X[z^L]

$$

  

#### step ii. filter

对插0后的信号进行滤波，z域上相乘：

$$

Y[z] = H[z]X_1[z] = H[z]X[z^L]

$$

这里$Y[z]$就是要得到的信号。

  

#### 步骤二合一：Polyphase

  

上述步骤的弊端是，步骤复杂，尤其是插0操作，假设原时钟是100M，L=8，则还需要7个八相800M的时钟用于插值，这个硬件开销是巨大的，常见的40nm工艺的数字综合都很难hold住800M。更不要说，项目正在做的原时钟400M, L=8, 需要的3.2G Hz的时钟信号了。

  

因此可以用下列的polyphase FIR intepolator Filter：

  

其原理如下，如果对$H[z]$进行展开：

$$

H[z] = \sum_{n=0}^{N-1}h[n]z^{-n}

$$

令$n = mL + r$ :

$$

H[z] = \sum_{m=0}^{\frac{N}{L}-1}\sum_{r=0}^{L-1}h[mL+r]z^{-(mL+r)}

$$

**我们的目的是，为避免使用需要高频时钟的信号，减小硬件开销，仅仅使用未经过插0的原信号 $x[k]$ ，仍旧得到下列式子**

  

$$

Y[z] = H[z]X_1[z] = H[z]X[z^L]

$$

代入新的$H[z]$

  

$$

Y[z] =\sum_{m=0}^{\frac{N}{L}-1}\sum_{r=0}^{L-1}h[mL+r]z^{-(mL+r)}X[z^L]

$$

  

同样，展开$X[z^L]$

$$

X[z^L] = \sum_{k}x[k]z^{-kL}

$$

  

`Chen SHU：对以下推导进行修改。我发现下午我的推导也有错误（错的很离谱hhh）。另外师兄你如果想进行这样的连等的话，可以尝试下\begin{aligned} \end{aligned}的方式，需要对齐的符号之前多些一个&就行，比如想让等号对齐就可以使用&=`

  

>代入

>$$

> \begin{array}{l}

> Y[z]

>=\sum_{m=0}^{\frac{N}{L}-1}\sum_{r=0}^{L-1}h[mL+r]z^{-(mL+r)}\sum_{k}x[k]z^{-kL} \\ \\

>

>\ \ \ \ \ \ \ \ =\sum_{k} \sum_{m=0}^{\frac{N}{L}-1}\sum_{r=0}^{L-1}h[mL+r-kL]x[k]z^{-kL-mL-r} \\

>\end{array}

>$$

`修改如下：`

$$

\begin{aligned}

Y[z] &= \sum_{k}x[k] \cdot z^{-kL}\sum_{m = 0}^{\frac{N}{L}-1}\sum_{r=0}^{L-1}h[mL+r]z^{-(mL+r)},\\

Y[z] &\overset{z}{\leftrightarrow} y[n] = \sum_{k}x[k]h[n-kL].

\end{aligned}

$$

  

  

由上式可知，我可以实现仅用$x[k]$和$h[n]$进行一些看起来复杂一些的运算，就能实现目的。不需要复杂的$x_1[k]$

  

这个复杂的式子可以用下图解释，假设$N = 20, L = 4$

  

![4a4e29adb1bba553fc0261bf03395fcb.png](en-resource://database/572:1)

最后在输出的时候，仍旧需要高频的时钟串行化，这部分可以利用模拟电路搭建，是可以实现的。推及我们项目要做的$N = 160, L = 8$, 其实现路线也是一致的。

`增加示意图`

![f0a535054eb937ded421a78f4b95f2ac.png](en-resource://database/573:1)@w=600

![1e46bc9e536e6f10b40129f7df75abc5.png](en-resource://database/574:1)

以上滤波器用方块简单代替，如果具体表示，其中一组sub-FIR的示意如下图所示，

![403522ec7d28a1a1d54e578aa52c05ef.png](en-resource://database/575:1)

  

  

# 三、滤波器验证方法

## 1. Matlab代码验证

由于电路中输入的是10-bit信号，其中1个为sign-bit，因此输入的范围是$[-512, 511]$，是没有直流分量的。

  

由于Matlab滤波器本身是单位能量的，所以输出的时域波形会是一系列小数。处理方法是，先把这一系列小数转换为10-bit signed 二进制数，再以这一系列小数为基准，转换为$[-512,511]$。以此便于对比前后波形。

  

首先，脉冲成型的步骤是必须要的，因此信号必须要通过Nyquist滤波器；同时，Nyquist滤波器本身又不太可能不具备上采样的功能，因为根升余弦让通带和阻带都在$f_s/2$附近。对于SRRC上采样滤波，得到的新信号。

  

## 2. 行为级验证

在matlab上，模拟加法树、乘法卷积以及加法器和乘法器的定点化操作，测试行为级的结果。

  

因为目前发现，不论是filter builder还是filter designer，在量化加法器和乘法器功能中，都不能很好地表征定点化对结果的影响。

  

目前发现，基于行为级的结果如下所示。

![a3f5debdc4e5a1b8d7bb896702707a7b.png](en-resource://database/585:1)

**能在远端压下40dB**

但是尽可能让它在多次行为级仿真中，压到-43dB，我才敢用它。

以下是记录并且测试的结果：

### 1）LPF_4phase_test1

![c4fc74dad2a318e43fb598aa1f9e1c6c.png](en-resource://database/586:1)

不可行，有时候只压了38dB

  

### 2）LPF_4phase_test2

![7c1036b9d1d5a9efdc6e5670d73e16e7.png](en-resource://database/587:1)

  

  

  

  

  

  

  

  

  

  

  

  

  

  

  

# 四、Verilog实现

  

## 1. 寄存器数量计算

八项FIR需要分出8路，每一路有12个$h[n]$，在定点化时，将系数设置为signed 11 en 10。存储系数的寄存器则需要

$$

N_h=8 \times 12 \times 11 = 1056

$$

  

## 2. sub-FIR实现结构

### (1) 直接型

![8eb586d577105ee64ed20a517f145f35.png](en-resource://database/578:1)

这是最直接的FIR，输入$x[n]$到得到$y[n]$之间有$N$拍的延迟（$N-1$个加法器和1个乘法器），需要的**加法器**、**乘法器**和**移位寄存器**皆为$N-1$。

不管是延迟还是需要的硬件资源都很多，不使用该方法。

  

### (2) 加法树型

加法部分用加法树实现，延时为$\log_2{N}$拍，需要的**加法器，乘法器，移位寄存器**仍为$N-1$，硬件资源仍旧要求很多，不采用。

![4ce47628ce29f0ff9434c5bd3cd741f2.png](en-resource://database/580:1)

  

### (3) 转置型

将**移位寄存器的位置调整**，其延时固定为2拍，即一个加法器+一个乘法的延时，但是移位寄存器会从原本的10增大为$10+14$，因为乘法器的输出位数会更高，而加法器乘法器的数量没有改变。所以这种方法的延时较低，但是硬件资源消耗也是最高的。

![423b8462f69308e5094eca35e17f3e48.png](en-resource://database/579:1)

  

### (4) 对称型

由于设计的impulse response是：

![image.png](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAABfoAAANiCAYAAAA5ZnjMAAAgAElEQVR4AezdB7gU1d348WNDBEQBUZqKCiqICNg7xhpb1Fheu7H9TWKMmmiMvhpN1CSaWPMao0ZjTywx1sSOBRsq2MWGjaYoSEdQ/s/36NnM3bu7d2/j3r37Pc+z7O6UM2c+c/dh5nfO/GaxadOmLQwWBRRQQAEFFFBAAQUUUEABBRRQQAEFFFBAAQUUqEiBxSuy1TZaAQUUUEABBRRQQAEFFFBAAQUUUEABBRRQQAEFooCBfv8QFFBAAQUUUEABBRRQQAEFFFBAAQUUUEABBRSoYAED/RV88Gy6AgoooIACCiiggAIKKKCAAgoooIACCiiggAIG+v0bUEABBRRQQAEFFFBAAQUUUEABBRRQQAEFFFCgggUM9FfwwbPpCiiggAIKKKCAAgoooIACCiiggAIKKKCAAgoY6PdvQAEFFFBAAQUUUEABBRRQQAEFFFBAAQUUUECBChYw0F/BB8+mK6CAAgoooIACCiiggAIKKKCAAgoooIACCihgoN+/AQUUUEABBRRQQAEFFFBAAQUUUEABBRRQQAEFKljAQH8FHzybroACCiiggAIKKKCAAgoooIACCiiggAIKKKCAgX7/BhRQQAEFFFBAAQUUUEABBRRQQAEFFFBAAQUUqGABA/0VfPBsugIKKKCAAgoooIACCiiggAIKKKCAAgoooIACBvr9G1BAAQUUUEABBRRQQAEFFFBAAQUUUEABBRRQoIIFDPRX8MGz6QoooIACCiiggAIKKKCAAgoooIACCiiggAIKGOj3b0ABBRRQQAEFFFBAAQUUUEABBRRQQAEFFFBAgQoWMNBfwQfPpiuggAIKKKCAAgoooIACCiiggAIKKKCAAgoosKQECiiggAIKKKCAAgoooIACCiiggAIKKKBAcwostthizVl9WLhwYYPqZz3aNn/+/PhK9SyxxBKhXbt2YfHFF4/zqTzNq++GqP+rr74KCxYsiO9ff/11rSpYhm0tueSS8cX3hm4vVd5S203bL/SOAdaUpZZaKu5rWo72Zktj9z9bVzV8NtBfDUfZfVRAAQUUUEABBRRQQAEFFFBAAQUUUKAFBZoicF2s+Q2pm2Dzl19+GQPvBJyfeeaZ8PTTT8fvBNx79+4ddthhh9CzZ89A0D8FndN7sbbkT6dtBO8//vjj8Oqrr4b3338/TJs2LcybNy92IFAf9Xfo0CH06NEj9O/fP6y33nphmWWWiR0D+fWV+72ltltX+0aPHh1GjRoVF9twww0DL9qaSvYzNryy09JyvtcWWGzatGkN6+6qXZdTFFBAAQUUUEABBRRQoBkE0gVlYy5y0oUSF66FSlNeRDVlXbSVEXDZ0XSF2u80BRRQQAEFFGjdAg8//HD8P50R3Y05p8nuJeccBNEJlG+77bbZWQU/s93Zs2eHTz75JLz77rth5syZcd2OHTuGDz/8MEyYMCEG9plOR8C6664b66dDYLnllgsrr7xynJ/az/ZLFdo1Z86c8Oabb4Zx48aFzz//PAbwCep36tQpBvfZzqxZs+KL7XLOs8oqq8SAP9tj9H9d28lvQ0ttN78d6Tvtwf2NN96IFnR6UPr06RPWXnvt6MwxGTt2bOzcSN7Dhg2LTpwLWuoWcER/3UYuoYACCiiggAIKKKBAiwlwMTx+/Ph4Idi9e/eS7eBCkYvBpZdeutZyjBybOHFiWGONNWrNZxvM69KlS7yYqrVyPSZwIcaFGhevvNKFcD2qqLEoF75vvfVWbHfnzp1rzPOLAgoooIACClSOwIgRI8Kyyy4b+P+80PkBwexCKW3Yw2Id/qwzffr0MGPGjDoD/WyTejgnIqDMyHJG1TNynukEl2nfOuusE8+9CEq//vrrcRnmcV7Dec4KK6wQOwcIXlNnsSA8dRLcpgPhhRdeiG3s1q1b6Nu3b7xbgHqWX375uN0pU6bE8yc6AwiCjxkzJnY0tG/fPi5DZ0Yxm/y/gHK2S6cF+0THw+TJk2MnRGO3m9+O9B0jOjCoHwdMunbtGmd/+umn8Tvt4bh88MEHcTk+cycFx4LzUwP9SbP0u4H+0j7OVUABBRRQQAEFFFCgSQW4SPviiy/iRSEXLoUudLMb5OL1xBNPDBtvvHE4+eSTs7NqfOYik9vNGYm2yy671AiyM+/ZZ58Nl1xySbjiiivi6Knsylzk/fjHPw4HH3xw2GeffbKz6v2ZDoMjjjgibLbZZuHUU0+NuVfrXUlmhXvuuSf88pe/DL/61a/CAQcc0Oj6MlX7UQEFFFBAAQUWscDQoUPDlltuGUdtp8A15ynpfCj7OTUtOy37mYA2AfAnnngiPP7442nxku+sw3kPgygIoq+66qoxuP/aa6/FukjXM2DAgDhv0qRJ4bPPPouBfQLR7733XiAwTRCaVD+Myi80uCI1gGVefvnl8Nxzz4W5c+eGIUOGxBd5/9NdCATb2SfOCekAWW211WLAn/1hoAMDHrbffvt4N0HySvUXey+1XeZhwOAQCvvVVNst1h6C9HSsEOTHc+DAgYG/A/abVEbMoxOIaTvuuGN44IEHYicEHSkUlrOUJ2Cgvzwnl1JAAQUUUEABBRRQoEkECPKfddZZ8QLm7LPPzo1oamzlXCgyMu2yyy6Lt0Qfe+yx8cJt6tSpcSTafffdFy8uubhj5BYXmFzc8c5FF+1affXVCzaDC7T8kVTp4pqLr+wFGBdsb7/9djj99NPjaLf8i1LWyxbmc0dBsUIwgNdVV10VNthgg3gbe6FlCRCwLylQUGgZpymggAIKKKBAywlwvsC5B6PmGZzAKHyCuZwHcB5DwHmTTTaJ73ynEBRn0AP583lPQXLOS6hn0003jXVmz0VK7SHnCdTNNhlNv+aaa8YgO8Fmgt/Uzwh/OgEIipPSh5z5vXr1iiPNP/roo3iuwfZZnxcB/EKF0fl0DnCOxTkMAW5G8XPuw34znYEYaeQ6+5DOr8hbT2CcuwFIMcQ5GncUlLOfpbbLtqmDVzp3Ypt0WnAe1Zjt5huwj3RUcFcEd0fQ2cGzB0jVwz6n/cWbjhbOH5mGFR0o3OVgqZ+Agf76ebm0AgoooIACCiiggAKNEuCilJFiZ5xxRryQYRQ9F5KNLVwQ/fCHP4wXahdccEGsbvjw4WHfffeNF8wpwH7bbbfFC1lGrF1zzTXx4pVbqbmQXXHFFWvcFs4FIC8eGscFWLYwAo6LNS7C6CigcNF79913xzyrXCgzEi1bmMZ2s6PfGBnHRW6pwsUtF9Ysx8VuocIFeQoOFJrvNAUUUEABBRRoWYF0XkHA/sknn4yB8+z5B50A5MTn//QU6KfFBIvfeeedGBjnfIYUgUwjvzupXVK95ewdgWTOuwhs807gfaWVVornQQTeOZfhRUcA9RLI55yH8xfaxWAJOigoBKipp1ign9H83D1AmprBgwcHUjDSmUAbCIKTu3/kyJFxJDv7nTogaNegQYMCgzU4z+J8iu2stdZasW2sX6qU2m52vVRP2ufGbjdbN3YpXc9LL70U9430kdyhyt0LqZDGiONOJwAdFBzX7bbbLvTr1y/uM+eOqZ1pHd+LCxjoL27jHAUUUEABBRRQQAEFmlyA0VKkx3nxxRfD5ZdfHi9QGT3FhSsXRLyyJY3IZzq3kOcXLh65CKLw/oMf/CAG9rn1m1FbjHS7+OKL44VqWpcRYqTCoXDxxEgrLj4J2Ge3z2grLnBZ/ic/+Un8zIUbF+jHH398DPTTWXDhhRfGUXXMS4UOjGzh4jXbuZDmcVF3yCGHxIs8LqBLld///vdFZ3Pxy0UhowEtCiiggAIKKNA6BTjv4HyBoDLvnMd873vfiwF3gusEtFNaGfaAz3QG7LXXXnHEOYHv22+/PeZ1T3XUFQhO5ycsx4tzBYLNpBvknIbzHQL+BLzTMryzHsF8Cnnl0zTaSPCfczQGSxQrjObnPIpgNudonJdRqJcX537sA0H/NC3NZ1u0i/M5BjtwjsbytLGuUmq7pdalDY3ZbrZu2skofoL8BPHpuFh//fVjZwnLsR0KxxdP7t7knTtDH3rooXj3w3e+851afw/Zbfi5toCB/tomTlFAAQUUUEABBRRQoFkFuKg9+uijw49+9KOYN/+iiy6KgXbS7px33nm5i8rUCEZ8jRo1Ko7AT9O4WGTU03/+8584kp0LJl50JOy3335xMW5z50KL9bOj6LngSoULXEZ+kd82LU/uWS6qL7300rD77rvHRbfYYotw5ZVXxgsucuZny/e///2Yqie7jex8Pmc7F/Ln8f03v/lNbluF5tc1jYtfLhAtCiiggAIKKNC6BThfIajPaHxeBIRJa0MaHc5ZOHch6EzhfIdzG0bdM7KdZXkOEHcaMgI/BYxL7TGBdB66S1CZwD0pcAjAk66HAQ7UT6Cf86pUX3rn3IK2cr7EoAWWo509evSI36m3WGE0P8F67jwgoM+5CoV9o035L9qRtst+00bWp510NCSTYttL00ttNy1T7L0x26VO9glHBpHw4pyTuxlI18PIfSyzhf3FhXn8DfCZ81IGgnCsVllllTiYhHqTTXZ9P9cUMNBf08NvCiiggAIKKKCAAgosEgHS3hx66KHhrrvuihczjKincPG4//7754L9XPDefPPNcUTXNttsk2sbo7sYqZ8Ko8p4rbzyyrkR/syjg4CRctmLQy50yTVL+eCDD8KYMWNiEJ+RVlxY8WBeXjwQrZzCxSsXaKVG5NNxUKqwLhe0+SVdXHOxl0bVpWUIBHDRzSg5LgAtCiiggAIKKND6BQjYci7CnXjc1XjttdfG7+lcgPMKAs4U/n9nWVL9MTqcdx5OS4CdAQrlBH/pEOBOAN4Zzc86vDjHIijNuQbnMQT103bZdnYQAel6WI6gPUFr2sT5R/65SVafNlInqRHZJwLwDKbgnIx1+c65DJ0IpEmkTrbPspwX0ibeWSY/QJ7dTv7nUtvNng9m12uK7VI3HSeco2bT9ZDaiHM89qHQ8WIa8+g84W+AY4IHKRs5RqRwpIPFUreAgf66jVxCAQUUUEABBRRQQIEmF2DUFilr/ud//qfGA3kZuXXggQfmpnEROGLEiJgm58gjj8y1g4vbFOjn4uxf//pXuO6668LJJ58ctt566zgin4XJW18sdQ/rpbz3XKhyEcYoNy5suVW81Aj9XEMyH7gYo7MhW7hYzeZizc7L/0zQngtpnmPABTCFCz3uIOCWfayyF6jMw2bbbbeNt8Xn1+d3BRRQQAEFFGhdAvw/zosAPqO9Odc44IADwqOPPhpT8nBXIoHxNFKe8wjOLUjXQwoclmUd1qWO7HlBsT0dPXp0TLEzYcKEeH7RrVu3OIqfADNBZdLicP7DoATOZZhO3ZwbMY3tEYRnWyxHAD49M4jztlKFwDWdC6zLCPfHHnssl66HAD718vwBUgCxXfab9pHmkbo5J8qO9C+1rey8YtvlHK9QaYrtsj9vvvlmHEBCfaTrGTZsWHQsFuRPbUnBfjpGuIsUdzoLnnrqqXiMhg4dWnBASFrf928EDPT7l6CAAgoooIACCiigQAsJcLHYFIWLtl133TXe6vzzn/88nHTSSbGzgLq56OJ27xQ4ZxoXsRQeZseFNReDKUDP8nQAcFFZ38KDeI877rh4ccYFLRev3GFAHv9yCrdqk8LnxBNPjJ0VrMPFMfXwTIM999yzxl0Dzz//fPjFL34R70bg4t+igAIKKKCAAq1bgP/TCaAPHz48DmrgvIOBBjxrJ3X0Z0ev85lzGB5Wy92ILEsQuX///jEgTl3Z5QvtPSl/ONdhZDh1MaqewHo6R2JEPZ0LnLsQ+Oe8iu+MROc8hG3Q4UCaIQZCUAfrsF3aXKwQvGZ/aS/nVYxYT89loj5Gv9MJwba4o5P2UCfBbrbBenRocF5Wn1Jqu+xjocK2G7vd5557Lg7QoN0E5jmm+FE3baqrsAz2rEMaHwzI2c8dAhwXznUtpQXqf/Zeuj7nKqCAAgoooIACCiigQJkC6aKn2EVXmdXExbhIPOecc+JFMwHw3XbbLV6wkhufz1xksR1GqPGZh8gxmp9OgO9+97u5QD+j2rgYZTRZQwq30x9zzDHx4ow8uldccUXZ1TBCjjsYRo4cGUdz0VamcYv8K6+8Em+550KdwoUzF3/c9j9gwICyt+GCCiiggAIKKNByAgSy+b+cPPt85m4+/k/n/3OC3Xxnejo3SoFvOgYIeBMcp7B8GunOMqUK5z2MEOfch9Q7jKAn2E/wmO0RmE4j9fnM+QepYji3YpusT2Ce4DV3KXL+Rhogtlsq0M86nGdNnjw5puBZY401YrtpK3cJTJo0KZ7fMCiC8ycc0rkhddNGUhVRaFO5pdR266qjMdvl/DM540egHr+0T3Vtm/ksyzqsS9oi7grgeHFOaaC/bkED/XUbuYQCCiiggAIKKKCAAs0iQJqaNIqMC5rGFi6uGOHORSqjobiwPOigg+KFERdIjEgjKM4oNoLpPOCM/P2M8CcNDhdW48ePjxdWjJhrSOHCjlz/XMRzYVyfwkg9Rs/xzADuMKAuLoS5MOaWfUZ08TA+CvO5A4ARfly0WxRQQAEFFFCg9QsQwCeYS4Cd844U0E9Bbr6naexNWp6R7dl5LJ8C8HXtdXoGEQMEOGfg3IFzJtL/pUJbOOegsB2+80qF8xoC/7wzKIFAPx0NpQL9DFYg0D9x4sQ4EIP9Tp0SBO7ZB7bBfjA9GbBN5rMNzuVoF9stt5Tabl11NGa7jMbHtnfv3uHOO++Md0CQXjG7X9ljm21L6gxgPq7cyckDl0nlxDltsfWydfg5hMKJmZRRQAEFFFBAAQUUUECBZhd4+umnY4odLmSaqnAxyEUnHQgExQn8c+s0I8K4pZp8r1tttVXMe8/ouH333TeOLuOhvHQ8EPBnxBsXwIuycHHINnlIMTlZ33vvvdzmaT+j2lI+XGbQTvLAbrzxxnEUXm5hPyiggAIKKKBAqxUgiMsAg0ceeSSMHTs2jpQnZQz/pz/44INxlD3LpMJnRt4zj2VYlhHrrEsd1MX3UoU7AxnAwCABBhNQX7qDgNH02UAy5yMUAs/pxXemsx3aw3TOmZjGqPNihVRBBPAZkc42UzCbdwL8vDOfQD+fs/OZRgcBgzPonKCTgWnllFLbTdvJf6depjVmu+TW55yNuyM4b+PuixSgp+4UxMeQDgVefObFPJah8Jl1qYOBMKRL2mijjcrZ9apf5r+/nKqnEEABBRRQQAEFFFBAgUUnwIUUtzhz0ZkuKtk6I9Z4WBwjwChc5DCNwD0Xe6mQX79QSSPCLr/88pj3le1wEczFE8Fz6k6jwriwPfXUU2PwnxFjzOfCmVvT0zKFttGU09JFK6P3KYzoZ+QXOWvJY8vFHiP9uUjnIXbsHxeGzOczy1sUUEABBRRQoDIECOpyXkP6QM43GAHO+Q3nIJzbcF6QPS/iM9M4f+H/fQLe5LpngALnAgSAqbNU6dq1awyWUwd3A3KOwzkGdzlSP+dDnFuk4HuhujiP4sU5Cu3n7kkGV9CeYoWc/rSRuyU//PDD2M5spwIdBwTxSQeULdztkM7LOP/bYIMN4j7jkILh2eXzP9e13WwdnGel0tjtcucEgXn2N3WKpLrxxY50RbxnC8tyHsh7ahvHlO8sS2cKg0Hy18vW4edvBEr/ElRSQAEFFFBAAQUUUECBZhGYMmVKHGG/6aabxtQ0aSOMut9ll13S1/hO0J9c+9dff31uegqQ5yZ8OxKLuwMY0c9DzLho4sLylltuCYcddljYa6+9sovHizEunLjoZPTV/fffH3hgHcsuqsJoNS7mGKFHIZ0QwXs6QQ444IC4LzwvgGA/F45YcFE6atSoOJqftD4WBRRQQAEFFKgMgRTIJYhLUJhA9j//+c/Qt2/f+P9+Cu6mvSG4y3kN5wSPP/54XJa0hKzLspRUZ1on/50ANucX1MW5Dp0KnCNxByNpDQnwE/Cvq5400pxzODodOHcpleqQBwbz7CM6MDiPo4Ng8ODBsaOBNnEOtueee8b94Dvbpy3chcndCqQsJMhNKhw6A9hmOaXUdlNgP72nfW6K7dI+AvqFCseKVET33ntvwC8NKMGEY7HPPvvEdxzyC20rd9/z16227wb6q+2Iu78KKKCAAgoooIACrUKANDRvv/12zD1KkJ3CRSsXgATkufCkENi+6aabArdhf+c734nT+IfRb3/9619rXFAxuuyGG26IdwlcdNFFMXj/1FNPhX//+99hk002ibc+pwrYJhenXOjRaUBwnfoYOV8oeM4y3EbNe6kRVVywcQHHKDfuRqircEcD+5Zy3HIhy4PXHnrooXjLOtO5kObWekb0p1vBybPLRSGdFBYFFFBAAQUUqBwBzj3o5Of5QQRxOfdI6XU4j+BcIwWimU/wmKA45yoEiDm34Q5E6kjLldp7gtmsT8CcNDAE3tk25xacr6Qge6k62A4v2sf5EB0DnLfRnkLBaeqifZxTEaR+5ZVXYs55OjbodCCPPi+C3NRJ3n/SATHinQA/HQTsO/PGjRsXOwNYlnOwFJwv1t5S2+Xcj3NAtk27Cb431XZpV7G2YcdxpRODNpCqkUK6RqZlj3n+fpWqN3/Zav9uoL/a/wLcfwUUUEABBRRQQIFFLsDFDPn5GamezTk6bNiwOHJ91113zY1w58Lv0UcfjR0Ahx9+eK6t3LKecramiVw80nlADn4uaCmsTz77I444IncxTOcBQfLf/va38QKVC2huC6dzgDsM0gNvU72807FwzTXXxIttLq65SC1UuCOBdEBcAJOPlsB9sdFdXPhyMctofSwojPgiv+tll10W273mmmvGi3DeceOil4A/F+m0tVjdhdrmNAUUUEABBRRoeQGCvpwncBch5y7f//73Y6PIZU8qHALo/H9P4f95AtKchzCdtD233357DIgTRKeuugqBYuph8ACDCThX4fyDuwo57yDYX25hUAXnUazDi/0oVmg351R0BrAO527sIx0FBLuzgX6C3QTcOb/h/InzrLQtguGcMzEYhMEPbDP5FNp2S223UFvStNQBQCcEdxxsv/32cRYupDeipGXSOr7XX8BAf/3NXEMBBRRQQAEFFFBAgUYJcBE3YsSIGFxndFkqBOjLLdxung38sx4XrFy4ku+WC18umLgwZjT/iSeemAv+c4dAtnBByB0GXFBy8UhAPT+AzsUnF6hcoDH6q1DhgpeH46Zb6lmGergoZ5/zCxf3XORzoZ4dmU/OXjpAGM2WytFHHx0/ckE4cuTImFuXC0WLAgoooIACClSOAOcnvDif4P9+zlW485BANoMD9t577zjinHMSCsF9zhceeOCBeMci67EO6/I51VeXAOc2LE+KIM6XuKOQfP2c96TBBHUFmlmW8xDOieg0YFBFXeuwXYLznOORd5/R+Zyb0QbOv1LQnv2gPs6ZOA+inYzmp2PgmWeeiQ8RZjQ/efDpIGC7pbbdUtut6zgwn04K2pc+l7OOy5QnYKC/PCeXUkABBRRQQAEFFFCgSQS4KGNkFheXBx98cC5FT1NUnm7t5lZ0CgF5Rq1xUbjZZpvFUWts/8EHH4yjxQjCc6HFSCpS/vDwtvvuuy9svfXWsRMiBfu5COUW6z/84Q/xIvScc87J3XGQbTfBeW6979evX62Oguxy6TNB/jFjxsRRXSlXK/MY0X/XXXfFC/K0bHqnI+PFF18Me+yxR+DhehYFFFBAAQUUqAwBzkE47yB1zPDhw+O5COcYBLAJehM8T6kL0zkIe8Y0gt8E2An8MwqfYDGBceqizlJBb+pgPsF0gu6MlifQz3OMuKOQ76nToFQ9tIlOBwYisB6dDaVG1qftso+0k3fOd8i7z6j+bBoe5rGfdHZQNyP+qZv2kLaQczyew8SgCjomqIP1ixXWa4ntFmtPdjptS26lvLPr+Lk8AQP95Tm5lAIKKKCAAgoooIACTSLASLD//Oc/8QFwpOppqsLFHg+wZZQ7qXC46GUEGB0KhxxySAy8c1HF7eCMImOkGBeABNvPP//8GFw/6aSTwoUXXhi/n3vuuTFgzzLkhKUTgItgXgceeGC8yM5ve8+ePQOvQoX2ZAsX9I899li8qOWOAy5iy8npT0cE6YnIsUvQP1u46E0pi7LT/ayAAgoooIACLS9AoJ07/Pj/m4A2rxToZR6fJ0yYkAvK02KmMY8UOIx2zy7PfOqiTpapq7Au50Kcq6Tc9+TE57yIOyyzKYMK1cW5FikROV9hUEIKxhdaNjuN7XLeQwcF+8D2Ob/KLyyX2sgdDnQscA5GJwfTSZ3IYBGmc75DfaVKS203tYljwn7yos2ljhHzWCYtX2rZVL/vtQUM9Nc2cYoCCiiggAIKKKCAAs0mwG3bPACOnPjZtD2N3SAXngT1yVvPiDAC+FdddVXYfPPNAx0K48ePjxenXEBzochIOpY5/fTT48Nzf/nLX8ZOAnL5n3DCCeG8884Lv/rVr+KD6liWVypc3NZVuJBmxBu33nPh9uabb8YLUj5z4clD6f71r3/F0fw8jPcvf/lLuPjii+uqNs5nNBttzC9//OMf44OM86f7XQEFFFBAAQVaXoDzEzrrGUlfKJDL+QGvQoXli63DuQZ1l1M4P2F0Pedg3AFJ0J7zJ4L2BPoJsOcXtsuABe4C4HyLz3Q8lDOiP1sX+8b2KfkDILLLpc8sy3lTSufDPnJXI4YE+cnZX05pqe2yj3SOYJoMi7WXZekMKWfZYnU4PQQD/f4VKKCAAgoooIACCiiwCAVIN3PAAQfEUWzlXpQyir7QyK9ss6dMmRIvQAcMGBBvbefBulyM/vrXv46jvv72t7/FwD8XWttss00coU9gnGD82WefHTsDqI9OgVNOOSU+UJfnCDB6v65tZ9uRPnPnAB0FjDzjFvRXX301ptthZBoXc0888US86Ntvv/3ixTnphc4444y0eoPeSRlkUUABBRRQQIHWKbDDDjvEc4AUzG2KVokrRuoAACAASURBVBLE5jyJgHh9Cil8eDAvKQTJg09QnfOyYgF4tsO5DedWjMjnbgQGHhRbvlhbqKfcwrKct7FvPKSYz3Rq8OBe7uIsN9DP9hb1djkm7733XryLFSP2gbta58yZE49XMqDDgmlPPvlkPF/lb4NlWZc6LPUTUKx+Xi6tgAIKKKCAAgoooECjBBgxduihh5ZdBxc5pM2pa/Q/o+JJu8OoL7ax5557xtu9CfxzYXjUUUfFFD4E7bndm/d99tkn5qolfz/LUHjfcccd43yC73UF+dkWo7XS+mnHuIDea6+94kNzmbbvvvvG0fuMfqPwfAJyzA4ZMiR+33LLLQMviwIKKKCAAgq0TQHuMmzOkh7gW9c2UuB51VVXjQ/I5W5HRsnnn8vk10OwnMELrEenAOdI9Q3059dZ13e2yTa424C7CNZdd93YOcHdoc1ZGrpd1iNQjxMpkbhbAlem856eq8B3Ct95vfvuu7ll0rbpUKGutGxz7m9bqXuxadOmld+V1Fb22v1QQAEFFFBAAQUUUEABBRRQQAEFFFBAAQUUUKCNCNR+8kMb2TF3QwEFFFBAAQUUUEABBRRQQAEFFFBAAQUUUECBahAw0F8NR9l9VEABBRRQQAEFFFBAAQUUUEABBRRQQAEFFGizAgb62+yhdccUUEABBRRQQAEFFFBAAQUUUEABBRRQQAEFqkHAh/G2sqP8ySeftLIW2RwFFFBAAQUUUEABBRRQQAEFFFBAAQUUUKDlBXr06BG+/vrrlm9IK2yBgf5WdFAmT54cTjvttNC+fftW1CqbokB1CcydOzfusL/D6jru7m3rE/C32PqOiS2qPgF/h9V3zN3j1ingb7F1HhdbVX0C/har75i7x61PgN/hJZdcEjp06ND6GtcKWrTYtGnTFraCdtiEEMIxxxwTXn/99bDjjjvqoYACLSRw//33xy37O2yhA+BmFfhWwN+ifwoKtLyAv8OWPwa2QAEE/C36d6BA6xDwt9g6joOtqG4BfoeDBw8Of/rTn6obosjeO6K/CExLTeaP9bzzzmupzbtdBapeYOLEidHA32HV/ykI0MIC/hZb+AC4eQVCCP4O/TNQoHUI+FtsHcfBVijgb9G/AQVaXiD9Dlu+Ja2zBT6Mt3UeF1ulgAIKKKCAAgoooIACCiiggAIKKKCAAgoooEBZAgb6y2JyIQUUUEABBRRQQAEFFFBAAQUUUEABBRRQQAEFWqeAgf7WeVxslQIKKKCAAgoooIACCiiggAIKKKCAAgoooIACZQkY6C+LyYUUUEABBRRQQAEFFFBAAQUUUEABBRRQQAEFFGidAgb6W+dxsVUKKKCAAgoooIACCiiggAIKKKCAAgoooIACCpQlYKC/LCYXUkABBRRQQAEFFFBAAQUUUEABBRRQQAEFFFCgdQoY6G+dx8VWKaCAAgoooIACCiiggAIKKKCAAgoooIACCihQloCB/rKYXEgBBRRQQAEFFFBAAQUUUEABBRRQQAEFFFBAgdYpYKC/dR4XW6WAAgoooIACCiiggAIKKKCAAgoooIACCiigQFkCBvrLYnIhBRRQQAEFFFBAAQUUUEABBRRQQAEFFFBAAQVap8CSrbNZtkoBBRRoGYGePXu2zIbdqgIK1BDwt1iDwy8KtIiAv8MWYXejCtQS8LdYi8QJCrSIgL/FFmF3owooUA8BA/31wHJRBRRo+wInnnhi299J91CBChDwt1gBB8kmtnkBf4dt/hC7gxUi4G+xQg6UzaxogY8++qjO9u+///5xmXKWrbMyF1CgwgRWXnnlCmtxdTbXQH91Hnf3WgEFFFBAAQUUUEABBRRQQAEFFKh6gQ8//DCccMIJoUOHDlVvIYAChQRmz54dLrzwwrDKKqsUmu20ViRgoL8VHQybooACCiiggAIKKKCAAgoooIACCiiw6AROO+208N5774Wdd9550W3ULSlQQQL33Xdf4Hdy/fXXV1Crq7OpBvqr87i71woooIACCiiggAIKKKCAAgoooIACIYRBgwaFc845RwsFFCggwF0vlsoQWLwymmkrFVBAAQUUUEABBRRQQAEFFFBAAQUUUGBRCkyePDnwKlYWLFgQ3n333TBr1qxii8TpU6dODTzf4Ouvvy66XLl1Fa3AGTkBjseYMWNqHBeOAccKZ0vbFHBEf9s8ru6VAgoooIACCiiggAIKKKCAAgoooIACDRYgKH/LLbeE8ePHh6OOOiqsuuqqYckla4YSv/jii/DjH/847LfffmHIkCE1ttW5c+fcOiNGjAj3339/+OEPfxjWXnvtsPTSS9dYli+prtNPPz1svvnmteYzYd68eWHcuHFhzpw5NeanbX311VcF5y+zzDJhtdVWK7jdGhW10BescZ4yZUpZLUj7m3880spPPPFEOPjgg8MDDzwQhg4dGieTgof0OzfeeGPo1q1bWtT3NiRQ89fZhnbMXVFAAQUUUEABBRRQQAEFFFBAAQUUUECBhgnMnz8/7L777uH8888Pv/jFL8LZZ58dg/SFavvd734XOnXqVGPWpptuGlMiLbfccmHYsGHhjTfeCCeffHI466yzwgYbbBDef//9MH369Nw6BPr5/tZbbwUC89my5pprxvpZ56CDDoqB/vbt28dFZs6cGYP4BLAJlOfPnzt3bqyP+WuttVa22oKfGfk+Y8aM0KdPn7D44ovHz9TLw2iXWGKJGussXLgwfPLJJ4H3Hj161JhXny90YJA+is6Qrl27llw1u7+FAvbs74MPPhgOPfTQMHDgwJJ1MZP6vvzyyzq3W2dFLtDiAgb6W/wQ2AAFFFBAAQUUUEABBRRQQAEFFFBAAQValwCj7hnF/6c//SlceeWVgVHijMYvVK6++uqio/BZnnpOPfXUOLr8lVdeCf379w/HHntsHH2fOghIKUMgP9tpkKYxGj2N8ie4fckll+SC9iNHjgy/+c1vcs3Knz927Nhw3HHH5ebX9eGuu+4Ko0aNCn/4wx8CnQmPPvpovLPhz3/+c1h22WVzqxPcnzRpUjjvvPNikJw7ERpbjjzyyPDd7363ZDUvvfRS+Mc//lF0GTpUHn/88XDGGWfUeQcDnQL33HNPvJuCuzbo2LBUroCB/so9drZcAQUUUEABBRRQQAEFFFBAAQUUUECBZhcgCEyZOHFimDBhQm57aRQ+wfQ0wj43M4Q40r5du3ZxPVLnpCD2Z599FhfLdhAw7cADDwzZ1D08H4CR6dkUNdxp8Nprr8WR6FTCtrMlf/4HH3wQmFZOIfUPdxSsvvrqcX8I5hM479evX+jYsWOuCkbgU+9VV10VrrjiinDKKafk5jXmw1//+tdwxx13lKyCEfh9+/YtuAwdI3TIcAfEVlttVXCZNJFln3zyyXDppZfG44JRoZRKaXnfW7+Agf7Wf4xsoQIKKKCAAgoooIACCiiggAIKKKBAlQu88PGMsOzSNVPH5JPMmPdVWL/Pf0ed58+vz3cCygT2CdCnQDuB7csvvzz07NmzRlWXXXZZje8Ekcmlz/KktCH9z/HHHx/z+NeVmiZbESPOCbJn0+KQVofR6qljIT/wnT8/pe7J1lvs8+zZs8PHH38cNtxww7hICuhvvPHGudHuBMSffvrpeKfDCy+8EDp06FCsunpPP/zww3OdIcVWZkQ/z04oVHjg8d///vf43ARSJhUrHB8e1stdCzvuuGM46aSTDPIXw6qg6Qb6K+hg2VQFFFBAAQUUUEABBRRQQAEFFFBAgeoU2OayF8MBw1YqufM3vTg5TD9365LLlDtz9OjRMUD/05/+NKbcSQH6Y445Jo66p54U0Cfwn1LwMJ0Hyx5xxBFxZDzLkK+fh/aSmob3VN58881cgDndHZCdxnLZVD4E9zfZZJPw85//PKYDYj6BbzoUyJXPQ3qZT6cCD7jlbgLeL7jgglzHQNp2emc+AfJPP/00vt5+++14t8Dzzz8f8/OTaogH2jKCf+WVV47PEWAUPHntSWt05513pqpqvNMBwb5wd0Cyq7FA5gspc0hv9M9//rNofZnF4z6mzpc0nQ4N1qcTImuc5qd3jgfHlrsQaBvH15H8Saey3w30V/bxs/UKKKCAAgoooIACCiiggAIKKKCAAlUgwEj9y/cunCM/7f7YT2anj41+J0hOgJ60PQTOf/SjH9Wqk+D8T37yk7DPPvuE9dZbL87v3r17ePXVV2Pgn9H4jCxnRPzOO+8cfv/73wfS8dApwINtCcATYOd7erAsdwxQCJR/+OGH4ZFHHonrE8ieNWtWTOWTgvKpQaT3IW9/796943xy/ZMCiA6IE088MbcOOfbzg+4E7OkouPXWW2Ngn1H95PxfbLHFYsfB559/HttMW0477bSw1FJLhe233z4+qLhLly7h7rvvTs2o8U4HxBZbbBFuuOGGmJKoxsxvv1D3e++9F79RJ69yCx0SFLzpgGCEPoH+zp07F62Chx3zTAPSDeH9v//7v/H4FF3BGRUlYKC/og6XjVVAAQUUUEABBRRQQAEFFFBAAQUUUKD5BQjO82I0Ow/DJcieCvn0CVAT6Of1xz/+MTeif88994yft9122xpBZDoCbrrpplgF6xOQvvnmm8P1118f1lhjjXD00Ufn0uMwyp6HxPKg2xVXXDGuQwfBOeecE1KAO7WFDoB33303LkcbCfivssoqsS5G6p9wwglp0fDrX/867LTTTrnvfKBD4+CDDw577LFHDNqzX3RqENAnKP6f//wnPkiYdrAsdxVwVwOFOwiKFTovSAGUOjAKLffcc8/FNESMsme7vXr1yjmyfHoeAtNTYX+5u4A7AGgL3tiR25+OguwDg9M66Z3R/qQHOuCAA8Jvf/vbGscnLeN75QoY6K/cY2fLFVBAAQUUUEABBRRoFoEZ8xaEybMWhK+/Xhg6dlkQll3ay4ZmgbZSBRRQQIFWK/BmE46Mb6qdJP9+frvWXrHp8sMXayeB8fzg+KhRo2oEqO+9996w+eab56ogQD1t2rTcdz4Q3GeUPqPPCWzzImUM6912223xroAUFJ86dWq49tprw9Zbbx2D36xPYPvGG2+sVeejjz4aLrroohjQ58G5BP1JS9O/f/9cx0GNlfK+kDaHh9fSHnLfb7bZZjE1Dp0N999/f/zONJarT6Fjg0B+qZJs04OI995779ydEazHw3kppEFKhSA/z0T485//HNZaa604+d///ne8M4IR+s8880xaNPfO8aAjgbsrDPLnWNrcB8/Y29whdYcUUEABBRRQQAEFFGiYwPgv5gVeL46fEZ5+9/Pw5ZfzwtYDFg/Dei8bei+3dHw1rGbXUkABBRRQoLIEfnjbm62uwTyMN79dj/5oWLO2k2A3KWsYJZ6C8ATnU4Ca0fwEoQmwMwI+ldtvvz1sueWWYeDAgWlSzD1PehkezDtlypSY654889wx8Je//CU8+eSTYbfddos59UnX8+yzz8bOhPz88alN5ORntD0Pnz3kkEPCrrvuGrbbbru4zv/93//FjgM6FdLo/lxD8j6k+giEk5efdQjQM3Ke94022ihOZ0R/XXXlVV3vr6Qyyo7I51kHFHLqp0K7uLMgWwjkk2s//wG8dKy8/vrr4eqrr47GPHiXtET5y2Xr8nPlChjor9xjZ8sVUEABBRRQQAEFFGgygec+nB6uemZCuPLZCTXq/PtrY+P3ozbuFY7cpFfYaJXieV9rrOgXBRRQQAEFKlhg7RU7trrWE+hf1O0iNc1ZZ50Vdtlll7DXXnvFB9DyANdUCBiTLobc9X369AnpAbE8EJdAfyqMlh87dmxchsA9aXgIaK+wwgox6Eye/7vuuisMGzYskLeekezk2E95/6mHYPbHH38c5s2bFwPXL7/8ctwe6X/SctR98cUXx/V5YC9tI7idOgtoYzYNDvWSo59A+HXXXRe3zTYYvU9AnQ4JOjFIL8RzCE499dSiD/VN+9qYd54nMHjw4FwVtItCup1UGNHPaP5sSSP+uTMgFbxIuXTmmWfGZxeQHqlQSR0ddB7k2xRa3mmtV8BAf+s9NrZMAQUUUEABBRRQQIFFIsAo/o0vfr7ktugA4PXxGZs7sr+klDMVUEABBdqCwJ/3/iYlSmvalzc/mRUWdbsYET5x4sQ4yp2gP0FwguqMNCdA3rFjxzgif//9948pYQjUE+xnPuumkeMEoHk4LbntCSy/9dZbsSMgzSefP6Pn//a3v8UOAdL0/M///E8NfkbvE8R/5513YjA85ZgnIP/mm28GHvxLhwI5/AcNGhRz9dMZsNVWW4UOHTrE+Yx6zwbN2QAB7oMOOijm+CcFEZ0WLP/CCy/EZwj86le/ip0S5NzPH0lfo4GN+ILZ2muvHR8KnK0mjejnTohsIS0R+flLFbx4yDDPJWCfSXtEh0V+SZ05dODQuWKpXAED/ZV77Gy5AgoooIACCiiggAJNInDm/ePKrodlr9x37bKXd0EFFFBAAQUUqFwBAs0Et1daaaVA3nyC/n379g2XX355HPFOjvi777477LfffjElz+9+97v4YF1y7r///vvh3HPPDcsss0wM/C9cuDCsu+66cdQ8aXkYxZ/KjBkzYpofHqTLiP2zzz47diKk+bwTrCafP0F5gtME4Cmsy0h9AtmMwGe5+fPnx+0yn44J0grdcMMNuZz2ccVv/2H0PoFzRr+vv/768QHELE8nAamHeCBxXUH1bH3pM+167bXXYgcDdy4UKxjz0GA6NvI7N6655pq42g9+8INaq3Ms6DShU6TQ8wNw4FVXoaOEugYMGFDXos5v5QIG+lv5AbJ5CiiggAIKKKCAAgo0p8D0uQvCVXnpekptj2X/uHu/0Lm9lxKlnJyngAIKKKBAUwvwf/brk2eVrJZlmrIwkpzAPoH+Bx54II5s79KlSxg3blzgwbGzZs0KPXv2DHvuuWcM/L/yyitx2pAhQ2JgnbQ3fGa0fufOncPyyy8fHn744RiIJ5UMAXtG4PPQWwL13BlA0DmNPOchufmB7K5du8bR+ulBtCNHjox559N+588nZdBxxx2XZhd8p3OBuwyGDx8eOwVSqiHuEmhIkJ+N0FGwxRZbRIcDDzyw4HaZyL5feeWVBedzBwWFugqVPfbYIxx//PG51ESFlslOowOEfcsWjgEPTs6mZMrO93PlCHh2XjnHypYqoIACCiiggAIKKNDkAiPenVrvOlln93W613s9V1BAAQUUUECBhgssveQS4bKR3wR+i9XCMk1VCAinlDgE9G+99daw9dZbB4L55H8nTc8TTzyR2xyjxwmqM+L+nHPOiR0EPESWADIP2uWhtgSUqefII4+MI/EJ0vMwXfL1k6Ofh91SCH7zbAAC5TvvvHMcFc+ofQrBajogyOVPefXVV+N7+id/Pg8TZlqpwnxS86SHB5OWiDsRGOFfqjCSnv3r3r32eRH7tOmmm8bnEBSrg0A+I+l5CG+hUmpEf1qeOwLojCk0qj8twzudLNOnTw8jRozIOTOdTgSOJ50qlsoWMNBf2cfP1iuggAIKKKCAAgoo0CiB0R/PrPf6rGOgv95srqCAAgoooECjBJ46rnTQuVGVF1h50qRJMU89o/UJ1BPs58G2d9xxRzj44INjsJjgMaPhScXDA28JqjMCn3zzBOgpBJcJQvOQWfLrM516CGIzkp00QNkH0LIO83kR8OcBuEcddVRI6WtIiUOKIALxFL737t07fk7fs/MZsZ4eEpxbKO8DQfnzzjsvN5Uc/WeccUbue7EP7PPJJ59ccDb79NRTTxWclyY++OCDRUfzp2V4/9nPfpb9WuMzo/p59kB64HCNmZkvdJrw4jkH+YVOlfxjkL+M31u/gIH+1n+MbKECCiiggAIKKKCAAs0m0JAUPA1Zp9l2wIoVUEABBRRQoFkEGDHPKHryyzNyn2AwueyzgW0Cx6Ti4QG2qTBan5H5KTDP9Ouuuy7Nzr0zP7tMbkbmQwr4p0kE99nmCSeckBuVzoh0HuJLML/QfDofLrzwwlzHQKqrNbyXY1CfdmLAXQmF7jDgwcc4WNquwGLTpk1b2HZ3r7L2jCeP84NMecgqq/W2VoG2IcCIBUqPHj3axg65FwpUqIC/xQo9cDa71Ql8MXdBmD73q9iuzu2XCMsVyKv/4vgZYf0LRtWr7S+cuGEY1nvZguuUs82CKzpRAQUKCvh/YkEWJyrQZAKMTKcYi2kyUitqYwKt6TeS2vKnP/2pjSk3ze44or9pHK1FAQUUUEABBRRQQIFWI/DRtLnhw2nzwujxM8Loj2fEdg3ts2wY2nvZsMryS4eVl2+faysBe76zTjmFZQsF+euzzXK24zIKKKCAAgoooIACCihQvoCB/vKtXFIBBRRQQAEFFFBAgVYvMPL9L8LVz04IVz83sWZbv/1++EY9w+Eb9wqb910uN//MHVYLR9zyRu57qQ8sm18ass38OvyugAIKKKCAAgoooIACDRdYvOGruqYCCiiggAIKKKCAAgq0JoEPp84NW1z6Qu0gf6aRdACwDMumcvjGPcMWqy0XVsmM9E/z0jvzWIZls6Wh28zW4WcFFFBAAQUUUEABBRRonICB/sb5ubYCCiiggAIKKKCAAq1G4KwHxpXdlvxlnzh2/XDmjqvFYP66PTvGesjpnwL8zGOZ/JJfT/787Pf6LJtdz88KKKCAAgoo0DICH330UeBVrCxYsCC89tprYfr06cUWidM//fTT8M4774Svv/666HLl1lW0gjY2oy77cnZ3zpw58fjwbmn7AqbuafvH2D1UQAEFFFBAAQUUqAIBHoJbK11Pif1m2Qu+17/GA3p/sFHPwIvc/ttcNjoM7d0pHLLBN9MKVdUU2yxUr9MUUEABBRRQoOUFCMrfc8894b333guHHnpoWGuttcJSSy1Vo2FffPFF+NnPfhZ23333sO6669aY16VLl9w6Tz31VPj3v/8dDjvssLDeeuuFZZZZpsayfEl1nX766WHzzTevNZ8JBKzfeOONMGvWrBrz07boLCg0v2PHjmHAgAEFt1ujolby5cMPPwyXXnppWH311cP/+3//L7bqgw8+CD169KjXPlDPiSeeGC655JJ4LPJ3jw6aadOmhVVWWSV/lt8rUMBAfwUeNJusgAIKKKCAAgoooEC+wIh3puZPqvM763xvUPday/HQXoL8FAL/xUpTbrPYNpyugAIKKKCAArUFps1dEL6YsyDOWG6ZJcPy7Zs+xDd//vywyy67xIDzr3/960AAPj+Yn1p2xRVXhOWW++/zf5i+0UYbxXXoHBg6dGgc0X/22WeHU045JWy88cbhzTffDFOn/vf8haAz31966aXw1Vdfparj+5AhQ0Lnzp0DgetjjjkmLFy4MHTo0CHOo4OAAPiNN94YpkyZUmv+7Nmzw2KLLRauv/76gsHuGhsKIXD3Ae3o169fWHzxxePnSZMmhf79+4cll6zpTDs+/vjj2J6mCJbTufLuu++Ga6+9Nhrgz3Ggg+Pcc88Nw4cPD7vttlu0yG93WnfixP8+p4m7AtiX559/PkyePDmu0rNnz7DGGmtEq9tuuy3MnTs3HH/88XFf8+v0e2UJ1PzrrKy221oFFFBAAQUUUEABBRT4VmDM+Jn1tmCdQoH+citqiW2W2zaXU0ABBRRQoC0KfDB1buA1ZvyMkP4fHtK7UxjSe9mwapf28dVU+7300kvHkd7nn39+DDy/+OKLRQP9//d//1d0FD7tIQjOyP/BgwfHQDaj60866aRAAD11EBDQfvvtt0O204BpdAjcfffdufoZvZ8doT5y5Mjwm9/8Jrfb+fPHjh0bjjvuuNz8uj5wFwP7yn63b98+PPbYY+Gf//xnuOyyy0KnTt8MhKAOAuvjx48Pv/vd72JHAx0hjSkE88eMGRP+9a9/hRVXXDF2knAMKATpscCMjo299947LpPdHvPvuuuu+ErT6eSg4+Ciiy7KdYzsuuuu4bvf/W649957A4H+Cy+8MNadtpXW9b3yBAz0V94xs8UKKKCAAgoooIACCtQSYDRffUtD1sluoyHrN2Sd7Db9rIACCiigQLUKEOA/8/5x4W+j/jtiO1qM+kbksA17xuftEPBv6kLqHgrpY95///1c9WkUPgFq0ubkFwL6BJAZib/22muH7bffPi7y2WefxfdsBwHTDjzwwBjgTql76Agg3U82ZdCXX34ZRo0aFTsJqOTVV1+tsdn8+YxqZ1o5hTsJ3nrrrZgyhyA/I/bpaGB0f7qDgHoYBU+nxFVXXRX+9re/xbsUyqm/2DIE7+nMOPPMM8P+++8fNthgg/DMM8/UWPyAAw6Io/N//OMfB0bt/+QnP6kR7MeZzhReqaROjtQxwnaeffbZ8Kc//SnMmzcvdmCsvPLKaXHfK1yg/lcDFb7DNl8BBRRQQAEFFFBAgbYoMLzf8vXerYask91IQ9ZvyDrZbfpZAQUUUECBahQoGuTPYKQOgDN3XK1JRvYTFE4B+hRov+6668LVV18dVl111dyWybd/6623xleayOhycuVfeeWVMRj9xz/+MRxxxBExZQ+j1cst1EO6INLNpEJ+fgLXKfBOO1daaaU0O+bvz85nVHu5hboZpb/pppvGVQjo07mx2Wab5VLb0CZG+V9++eWxEyDdkVDuNgotR2cEnQZ9+vQJTzzxRLj//vujH9+z+8a6W2+9dbj55pujy7777hurw4DOlvySUvekjhE6RY499tj4TAU6DHj+Ai8KnTL1OTb52/J7ywsY6G/5Y2ALFFBAAQUUUEABBRRotMCQXsuGvl3ah/enzi2rLpZlncaUlthmY9rrugoooIACClSqwJn/GRf+9nzeSP4COxOD/QtDuGb/AQXm1m8SufIvuOCCcPjhh8cAfQo48z2lqUkBfQL/2YA3AeajjjoqjoxnlPxWW20VfvWrXwUC09k0OqNHj45pY2gZdwd8/vnnITuN6f/7v/+bq5vgPoFu6kgj0Qle0wFBgJ6R+8z/4Q9/GGbOnBlTzMosNAAAIABJREFU77Rr1y5w50DqGMhXIAXPO++8EyZMmBC4q4AR/NxJMGLEiDBjxowYQCcIzkh/RvbTToLydH6QEuf222/PrzJ+J/jOvrBusstfMPmx34zmTwW/s846K3z/+98P2223XZqce6djhU4Y2sjzD1g+u35uwRDi8wnwoAwcODDm+Wcffvvb32YXC6eddlrBbdVYyC+tWsBAf6s+PDZOAQUUUEABBRRQQIHyBRjBd9jf3yhrBZZtitIS22yKdluHAgoooIAClSIwbc6CsoL8aX/oELhwj/5h+Qak9Ut18E5gnAA9D8898sgjY6qY7Hw+EzA++eSTA3nfBw0aFGf36tUrjkYn8M9DX3mILqlodthhhxgYJ5hO3TzYlofOEmBnWYLhHTt2jIFzgucEykmP8/DDD8cH+zIyn1z1PKCWvPO8Utl5551j3Wyb+QT/efAvD+ll5Hpah6B/ftCdzoG///3v4Y477gjTpk2LHQR0DPAgXjoLeDgvgX3msa/UQY57tkm7Wa9QoQ3bbLNNuOGGG2JKokLLkJef+Yy4p6TAP2mDsOP5A7zoLGF/2Xbfvn3jsq+//nrMs3/ppZdG+0cffbTWJvB64IEHAndUEOT//e9/n+sgqbWwEypewEB/xR9Cd0ABBRRQQAEFFFBAgW8EDt2wZ7j2+Ulh3OdzwvufFx7Z37dr+7Ba12UCyzZFaYltNkW7rUMBBRRQQIFSAjzstrWUEe9Oq3dTRrw7NewxqHu918uuQHCeFw/QffDBBwNB6VQIIBNoTqPwr7nmmtyo+9122y0G8hmJTpA/FToCCJhTCPYvu+yy4dxzzw033nhjTAV0zDHH5NLjMMqeAPrFF1+cC8yzzl/+8pcaAX7qogPgtddeC717944j8GkLI+95cC6BeoLbqZx66qm5+tI0gufkv999993jg3BJ1UPnAB0Rjz/+eNx31uvevXsMtBOE564GStYk1Zfe6QQg0J/fsZDm847PeeedFycR5CcvP0H5LbbYInaskHefZyLQGcFDej/55JP4QN5SdVIZdXF8brnllmhMp8FOO+0U71zg7oX8MmzYsNzxy5/n98oRMNBfOcfKliqggAIKKKCAAgooUKfAIz8cGq4dNTEG/KfOmR/GjJ8ZR/Qxqi8G+Dfo0WRB/tSYlthm2rbvCiiggAIKNIfACXe+3RzVNqjOMRNm1nu9l8bPbHSgP22UgH1++phXXnklBpAJKDPS/K677grpAbqsR3CatDfZQucAo+L79+8fA9GsSyCb9RhRv/fee+dyxE+ZMiWO9v/Od74TR+VTD6l6yPmfLdRJPns6EH7+85/HtDsEuHkg7TrrrJPrOMiuk/+Zkft0DNAeOh3Iz08HB6Po77nnnvh94403LquubN10bDzyyCPZSUU/s+38ID9W7MuTTz4ZR/HTEUJ6Hu4AIKVPGtmfXykm1EWHB8eha9eu8RkHPDS4WOGZBnRMWCpbwEB/ZR8/W6+AAgoooIACCiigQC0BRtnzGjNhRtjmstFhSK9O4Qcb9gqHbNij1rJNNaElttlUbbceBRRQQAEF8gWG9m7cc2zy62vMdzrqrxlVd37+7DYam7Yn1cXIetLnLL/88rmR6Tx8NwX/GdFPLv4XXnghzJs3L60W7rvvvrgMD9JN5e677w50EDCCnUA+HQHkuSfHPIF6HkK75557xgA7dxAwSp8R/3QGZEtqEzn5SetDoP/AAw8MO+64Y8w/f84558T6SK+z2mqrxY4FgvnFSqqPHPzPPfdc7GwgQM/+PP3002H99dePufC5Y4BOilJ1FdtGqemkKCLtDiP2eb4AQXzS9fAwY3Lx8xBj9o/jQLD/r3/9a7j++uvj/tKZkJ49QD0ch2effTZ2vNBxwjMRSN1DID97d0Wp9jivcgUM9FfusbPlCiiggAIKKKCAAgqUFOBhuQT5Kc0Z5M82oiW2md2+nxVQQAEFFGgKgQu+178pqmmSOui4r2+gf+t+yzfJtkmLw0NbScdDEJ7g9+qrr56rm+AxAfYzzjgjTidwT2FE/GabbZZbjhHrPOSWUfkE7hl1TufBCiusEAPQxx57bPjnP/8ZSCFDJwCBbB4Om/L+UxGj0997772YxoY7APjM9hipnpajblL1sP75558fttxyy7DJJpvElDvUQdvzR8LTNlLcMFKe1DiMpL/55ptjSqCJEyeGjz/+OI7sZ//J0Z/f8ZDbyQZ8IDjPfl9xxRUBg549e8bAPNvnuQf//ve/40N3CdazzzxrgOPBnQcE72kTHREUOguoh84V9qdLly6xTjoonn/++YKt466HutIAFVzRia1SwEB/qzwsNkoBBRRQQAEFFFBAAQUUUEABBRRQQIEQ6ERnVD/P4CmnsCzrNEUhx/2kSZNigJ6gP0FvRuCPGzcuBv3Js0/aG0aO77HHHjHNDcF35pNTP40i5zMPp2XEOiPoCfoThE/zmU4wmnQzjFxfc801Yyqf7D6Qwuayyy6LdwWsvfbagbz5rE+nwYIFC2KqHoL2bIeH8jKfuwIY2U++fgLgP/rRj2oF+mkvOfpJcUNgnQ4GRsnzgFyC62eddVZcnzpSR0a2XY35/NFHH8W7BgjMc0cBdzAcdthhMZBP20eMGBHuvPPOMHbs2ED6IDoq8CGH//Dhw8NvfvOb3HMCuPuAeuhMoXC3BQ8jZoQ/r2zBifp55gF3EFjahoCB/rZxHN0LBRRQQAEFFFBAAQUUUEABBRRQQIE2KnDWjquFMx8YF977rHSwf/Vuy4Qzd1ityRQIRPPwWUZ9f/755zHoz4h40scQXCeVDSl5SC1DEJ7AMkF6UvcQzGd0PUFz6lm4cGEMxjNinxQ5J5xwQq6dU6dOjcF5Av084Hb//fcPpAjKloMOOihsuOGG4eCDD44PymV7FEbgd+vWLQbHqZuR8TxIt2PHjnE+OfdpK3WvtdZa2SrjZ1LxkOKHB94yOp5tsD5pcAYOHBjvTmjIKH7aRecFdxvgUqgwj1H4dKLQkXDKKafExa6++up4FwH7wCh+HkqMEZ0YN910U9yPI488Mq6b6sUnW+gE+d3vfpedlPtM5w0dCqlTIDfDDxUtYKC/og+fjVdAAQUUUEABBRRQQAEFFFBAAQUUaOsCB2/wTaC4VLA/BfnTsk1h8tJLL4VVV1015q2/9957Y7odHtrKA3gZBZ9GjZNmhrQ+jDwnIE1KmGuvvTaOGid4TmCfVDKsy6h5guuMTifATd5+cu3z4FvuDGCE/h133BGD8wTa6TjI5sWnHtLWpKA9+ewZ2Z5K/nzadNxxx6XZBd9pOyPqt9122xjkp2OA9bhboSFBfjbCiPntt98+pgSiI6RU4Y4HgvukI6Jg849//COQdod28fyCMWPG5B4OzF0KjMpvaNuok44XUidZ2o6Agf62cyzdEwUUUEABBRRQQAEFFFBAAQUUUECBNiqQAvjXvzApfD5nfpg6e0FuT9fotkw4eP0eIS2Tm9GIDwSSeSAsgXYenHv77bfHdDEEnAlMk8Lnsccey22BEeUExxlxTwoagtV0FBAsJ+0OeftZj4D+0UcfHXPxsz4P0yVfP2ln+vTpE+vjYbiMRidnPyl+BgwYEOtjJiPvn3rqqZi7nu8E1LMlfz7phphWqnz11VexM4MOCgrLc5cAo/tLFTogaFuhgDn7RKCfvPt1FUbWk0aHgjudH9wR8fLLL8cg/4svvhj97rrrrly6o7rqzNaFe7aQ43+NNdYo2O7scn6uLAED/ZV1vGytAgoooIACCiiggAIKKKCAAgoooECVChDI5/XShJnh5QkzcwpNGeBPlfIg2tGjR4d99tknPP7443EEOIFrAvWHHnpoDMoTDCfVDoH5du3axRz+pMEhhz4P8KXMmjUrjuQnaE/Qnenf+c53YqodHjp74YUXxjsA0nZ5Zz6vFPBne3QcpPquuuqqXGoeUuSQXz8Vtpedz3eC56UKaXPIxZ8K6YZ48G5dhRH1P/3pTwsuRqcBD9Etp7APPBOAgiPpkAjyU0jnUywFT1ygxD8cm9tuu63gw3hxT89IKFGFsypIwEB/BR0sm6qAAgoooIACCiiggAIKKKCAAgoooMB6vToFXs1ZGMVPepzu3bvH1Dunn356HAWeDWxvttlmsROAVDqpHHLIIbETIJuu5vLLL0+zc+/Mzy6Tm5H5kAL+aRI567fbbrv4UN00+p8R/YxQ50G5hebTuUA+/5SzP9XVmt7Hjx8f/vCHP8QmkerojDPOaJIgPIF87q6wVIeAgf7qOM7upQIKKKCAAgoooIACCiiggAIKKKCAAmULMCL9z3/+c1y+WI77RR1IJrifH7imnWlaofYUWqdshEW0IPtACiOLAo0RWLwxK7uuAgoooIACCiiggAIKKKCAAgoooIACCiiggAIKtKyAgf6W9XfrCiiggAIKKKCAAgoooIACCiiggAIKKKCAAgo0SsDUPY3ic2UFFFBAAQUUUEABBRRQQAEFFFBAgUoW+OKLL3IPPq3k/bDtCjSHAL+P5ZZbrjmqts4mFjDQ38SgVqeAAgoooIACCiiggAIKKKCAAgooUBkCgwcPDrfeems4//zzSzb466+/jvMXX9zkGCWhnNnmBCZNmhS23HLLNrdfbXGHDPS3xaPqPimggAIKKKCAAgq0KYHPZ88Pn89eEPepa4clQ9cOS7Wp/WNnqmEf29xBc4cUUECBNiBw0kknlbUXM2fOjMt16tSprOVdSIG2IkBnWLm/k7ayz5W6Hwb6K/XI2W4FFFBAAQUUUECBNi/w9qezw9tT5oRXJs6ML3Z43Z6d4qv/CsuE/t07VLxBNexjxR8kd0ABBRRo4wLlBDEZ1Uzp0aNHG9dw9xRQoFIFDPRX6pGz3QoooIACCiiggAJtWuC+Nz4LN704Kdz44uS8/fzm+4HDVgoHDOsRdh7QLW9+5Xythn2snKNhSxVQQAEFFFBAAQUqWcBAfyUfPduugAIKKKCAAgoo0CYFGOW+y1Uvldw3OgB4vXXKJhU5sr8a9rHkAXSmAgoooIACCiiggAJNKGCgP4M5Y8aM8OGHHwaeJr3EEkuE7t27hz59+oR27dpllqr9cf78+eH1118PKV9b7SVCWHrppcPAgQNDhw6Vf3t1of1zmgIKKKCAAgoooEDTCZx5/7iyK2PZGw9ap+zlW8uC1bCPrcXadiiggAIKKKCAAgq0fQED/d8e46lTp4YHH3wwPPHEE2HWrFlxKnnXdt5557DRRhuVDPbPnj073HDDDWHcuNoXZF9++WX45JNPQq9evcIFF1wQ+vbt2/b/qtxDBRRQQAEFFFBAgQYLfDZ7frhpdH66nuLVsewle60ZulXQA3qrYR+LHzHnKKCAAgoooIACCijQ9AIG+kMIBOMffvjhcMstt4TtttsubL755nFU/z333BOuvPLKsPzyy4dBgwYV1We0/q677hrmzJlTY5mFCxeGl156Kdx3331hwIABoXPnzjXm+0UBBRRQQAEFFFBAgXyBEe9MzZ9U53fW+f7gFetcrrUsUA372FqsbYcCCiiggAIKKKBAdQgY6A8hTJgwIdx+++1hyy23DIcddlho3759PPqrr756OO2002Kgvn///jH9TqE/C5bfeuuta8369NNPw0MPPRRWXXXVcMghh4SuXbvWWsYJCiiggAIKKKCAAgpkBV6d+M3dpdlpdX1mne8Prmup1jO/Gvax9WjbEgUUUEABBRRQQIFqEFi8Gnayrn186623YrB/s802ywX5WWellVaKaXuef/75QNC+PoW8/Y888kgc0b/33nuHfv361Wd1l1VAAQUUUEABBRSoUoFuHZeq9543ZJ16b6QJV2hIexuyThM22aoUUEABBRRQQAEFFGjVAlUf6Ce9Drn1GW1PTv5s4YG8jOqfOHFimDRpUnZWnZ95qO/dd98dhg4dGrbaaqv4cN86V3IBBRRQQAEFFFBAgaoX2Lrf8vU2aMg69d5IE67QkPY2ZJ0mbLJVKaCAAgoooIACCijQqgWqPtC/YMGC8Pnnn4flllsudOzYsdbBogNg3rx54bPPPqs1r9iENJp/8uTJYffddw9dunQptqjTFVBAAQUUUEABBRSoIbBuj05hre4dakwr9YVlWaeSSjXsYyUdD9uqgAIKKKCAAgooUPkCVZ+j/6uvvgqzZs0Kiy22WHzlH9LFF/+mL2Tu3Ln5s4p+Z/T/iBEjYtqf9dZbr+hyhWZMnTo1vPjii4VmFZ22zjrrFJ3nDAUUqJ8AHXuU9F6/tV1aAQWaSiD9BtN7U9VrPQpUisAvh/cKh936TlnNZdlSv5Wvv/461lNqmUIbauh61FXOuk25j4Xa7zQF2opA+u2m97ayX+6HApUmkH6D6b3S2m97FWiNAq+99lq9mkXc1AHVxcmqPtBfnKZhc7ioIVA/fvz4cOihh4bOnTvXq6K33347XHjhhfVa5/zzz6/X8i6sgALFBb744os4c+mlly6+kHMUUKDZBfwtNjuxG2jlAjv2bRd26r9sePfz+eHtzwoPOOnfrX1Yo+tSgWW56ClWvvzyyzir1DKF1m3oetRVzrpNuY+F2u80BdqKgP8ntpUj6X5UuoC/xUo/gra/NQrUNwZK3HSjjTZqjbvSKtpU9YF+RvIvtdRSgVz9hUqa3q5du0Kza02bPXt2GDlyZOjTp08YNGhQrfl1TaBXav31169rsRrzDUjW4PCLAo0SSL+n9N6oylxZAQUaLJB+g+m9wRW5ogIVLHDTfv3C3W/PCne8NjVMmfVleHLcF2GFjkvF19ordgx7rtMl7Na/durJ/F1Od6jW9/fU0PXYfrnrNtU+5u+z3xVoSwLpt5ve29K+uS8KVJJA+g2m90pqu21VoLUK1DcGSqDfUlyg6gP9Sy65ZLzlY+zYsbmRR1kuAvc8lLfc20LIy//666+HbbbZJnTv3j1bVVmf+/fvH44//viylnUhBRRoeoF0G2a5v/mmb4E1KqAAAv4W/TtQ4BuBQzbqEg7ZqE94ddLMsM1lo8OgHh3DMZv2DvsNXalsojRgpb7/tzV0PRpWn3WbYh/LxnBBBSpQwP8TK/Cg2eQ2KeBvsU0eVneqhQXqGwN94YUXWrjFrXvzVf8wXoL4PXv2jA/knTJlSo2jxWj+jz76KAb5u3XrVmNesS/jxo2LD+4lNz93ClgUUEABBRRQQAEFFGiswKAenWKQn3rqE+Rv7HYX5frVsI+L0tNtKaCAAgoooIACClSXQNUH+jnca665Zjzqb7zxRliwYEHuL4D8ay+//HIYMGBAWHHFFXPTi30gPz93Biy//PJh1VVXLbaY0xVQQAEFFFBAAQUUUEABBRRQQAEFFFBAAQUUaDKBqk/dg+Rqq60Wttpqq/Dwww+HlVdeOay++uoxjc/TTz8d3nnnnXDEEUfkHqr71VdfxWA+nQCDBw8OHTv+Ny8qt3F9/PHHoVevXqHcOwCa7EhakQIKKKCAAgoooIACCiiggAIKKKCAAgoooEBVChjoDyEG8ffZZ59w2WWXheuuuy4MHDgwzJw5Mwb06QDYYostAg/tpcyfPz/cdNNNcaT/RRddFDsF0l8O86ZOnRpWWGGF0L59+zTZdwUUUEABBRRQQAEFFFBAAQUUUEABBRRQQAEFmk3AQP+3tP369Qs//elPwwMPPBDef//9+ACxPfbYIwwfPjw3mp9Fyem/zjrrhE6dOtUYzZ/mrbvuuqFr167m52+2P1krVkABBRRQQAEFFFBAAQUUUEABBRRQQAEFFMgKGOj/VoMR+3379g1HH3101qfWZx6wu//++9eazgTS+Pz4xz8uOM+JCiiggAIKKKCAAgoooIACCiiggAIKKKCAAgo0h4AP420OVetUQAEFFFBAAQUUUEABBRRQQAEFFFBAAQUUUGARCRjoX0TQbkYBBRRQQAEFFFBAAQUUUEABBRRQQAEFFFBAgeYQMNDfHKrWqYACCiiggAIKKKCAAgoooIACCiiggAIKKKDAIhIw0L+IoN2MAgoooIACCiiggAIKKKCAAgoooIACCiiggALNIWCgvzlUrVMBBRRQQAEFFFBAAQUUUEABBRRQQAEFFFBAgUUkYKB/EUG7GQUUUEABBRRQQAEFFFBAAQUUUEABBRRQQAEFmkPAQH9zqFqnAgoooIACCiiggAIKKKCAAgoooIACCiiggAKLSMBA/yKCdjMKKKCAAgoooIACCiiggAIKKKCAAgoooIACCjSHgIH+5lC1TgUUUEABBRRQQAEFFFBAAQUUUEABBRRQQAEFFpGAgf5FBO1mFFBAAQUUUEABBRRQQAEFFFBAAQUUUEABBRRoDgED/c2hap0KKKCAAgoooIACCiiggAIKKKCAAgoooIACCiwiAQP9iwjazSiggAIKKKCAAgoooIACCiiggAIKKKCAAgoo0BwCBvqbQ9U6FVBAAQUUUEABBRRQQAEFFFBAAQUUUEABBRRYRAIG+hcRtJtRQAEFFFBAAQUUUEABBRRQQAEFFFBAAQUUUKA5BAz0N4eqdSqggAIKKKCAAgoooIACCiiggAIKKKCAAgoosIgEDPQvImg3o4ACCiiggAIKKKCAAgoooIACCiiggAIKKKBAcwgY6G8OVetUQAEFFFBAAQUUUEABBRRQQAEFFFBAAQUUUGARCRjoX0TQbkYBBRRQQAEFFFBAAQUUUEABBRRQQAEFFFBAgeYQMNDfHKrWqYACCiiggAIKKKCAAgoooIACCiiggAIKKKDAIhIw0L+IoN2MAgoooIACCiiggAIKKKCAAgoooIACCiiggALNIWCgvzlUrVMBBRRQQAEFFFBAAQUUUEABBRRQQAEFFFBAgUUksOQi2o6bUUABBRRQQAEFFFCg6gUmzfgyTJr+ZXTo0bld6LFsu6o3aSyApo0VdH0FFFBAAQUUUECBtiBgoL8tHEX3QQEFFFBAAQUUUKBVC4wZPyOMnjAzjP1kVnhz8uzY1rVX6hDWWrFjGNqrUxjSe9lW3f7W2DhNW+NRsU0KKKCAAgoooIACLSVgoL+l5N2uAgoooIACCiigQFUIXDNqYrjzlU/Dna9NqbG/d772zdfvrbNC+N663cMPNuxZY75figtoWtzGOQoooIACCiiggALVKWCgvzqPu3utgAIKKKCAAgoosAgERo+fEQ7/+xslt0QHAK8hvTqFoY7sL2nFTE3rJHIBBRRQQAEFFFBAgSoU8GG8VXjQ3WUFFFBAAQUUUECBRSNw1v3jyt5QfZYtu9I2uGB9nOqzbBukcpcUUEABBRRQQAEFqkjAQH8VHWx3VQEFFFBAAQUUUGDRCUycPq9Wup5SW2dUP+tYigtoWtzGOQoooIACCiiggALVLWCgv7qPv3uvgAIKKKCAAgoo0EwCj707rd41N2Sdem+kgldoiE9D1qlgIpuugAIKKKCAAgooUKUCBvqr9MC72woooIACCiiggALNKzD2k9n13kBD1qn3Rip4hYb4NGSdCiay6QoooIACCiiggAJVKmCgv0oPvLutgAIKKKCAAgoo0LwCPTu3q/cGGrJOvTdSwSs0xKch61QwkU1XQAEFFFBAAQUUqFIBA/1VeuDdbQUUUEABBRRQQIHmFdi6X5d6b6Ah69R7IxW8QkN8GrJOBRPZdAUUUEABBRRQQIEqFTDQX6UH3t1WQAEFFFBAAQUUaF6Btbp3CMP6LFv2RliWdSzFBTQtbuMcBRRQQAEFFFBAgeoWMNBf3cffvVdAAQUUUEABBRRoRoGzdlit7Nrrs2zZlbbBBevjVJ9l2yCVu6SAAgoooIACCihQRQIG+qvoYLurCiiggAIKKKCAAotWYNd1Vgj/b9PeYf0SI/uZxzIsa6lbQNO6jVxCAQUUUEABBRRQoPoElqy+XXaPFVBAAQUUUEABBRRYdAKX771WuOf1KeGe1z8LE6bPC3e/NiX06twu9Oy8dNhg5c5h14Hdwq4DDfLX54hoWh8tl1VAAQUUUEABBRSoBgED/dVwlN1HBRRQQAEFFFBAgRYVIJDP661PZ4cXPpoe1uzeIfx8+CphFwP8DT4umjaYzhUVUEABBRRQQAEF2qCAqXva4EF1lxRQQAEFFFBAAQVapwABfl4Ug/xNc4w0bRpHa1FAAQUUUEABBRSobAED/ZV9/Gy9AgoooIACCiiggAIKKKCAAgoooIACCiigQJULGOiv8j8Ad18BBRRQQAEFFFBAAQUUUEABBRRQQAEFFFCgsgUM9Ff28bP1CiiggAIKKKCAAgoooIACCiiggAIKKKCAAlUuYKC/yv8A3H0FFFBAAQUUUEABBRRQQAEFFFBAAQUUUECByhYw0F/Zx8/WK6CAAgoooIACCiiggAIKKKCAAgoooIACClS5gIH+Kv8DcPcVUEABBRRQQAEFFFBAAQUUUEABBRRQQAEFKlvAQH9lHz9br4ACCiiggAIKKKCAAgoooIACCiiggAIKKFDlAgb6q/wPwN1XQAEFFFBAAQUUUEABBRRQQAEFFFBAAQUUqGwBA/2VffxsvQIKKKCAAgoooIACCiiggAIKKKCAAgoooECVCxjor/I/AHdfAQUUUEABBRRQQAEFFFBAAQUUUEABBRRQoLIFDPRX9vGz9QoooIACCiiggAIKKKCAAgoooIACCiiggAJVLmCgv8r/ANx9BRRQQAEFFFBAAQUUUEABBRRQQAEFFFBAgcoWMNBf2cfP1iuggAIKKKCAAgoooIACCiiggAIKKKCAAgpUuYCB/ir/A3D3FVBAAQUUUEABBRRQQAEFFFBAAQUUUEABBSpbwEB/ZR8/W6+AAgoooIACCiiggAIKKKCAAgoooIACCihQ5QIG+qv8D8DdV0ABBRRQQAEFFFBAAQUUUEABBRRQQAEFFKhsAQP9lX38bL0CCiiggAIKKKCAAgoooIACCiiggAIKKKBAlQsY6K/yPwB3XwEFFFBAAQUUUEABBRRQQAEFFFBAAQUUUKCyBQz0V/Y1NmtxAAAgAElEQVTxs/UKKKCAAgoooIACCiiggAIKKKCAAgoooIACVS5goL/K/wDcfQUUUEABBRRQQAEFFFBAAQUUUEABBRRQQIHKFjDQX9nHz9YroIACCiiggAIKKKCAAgoooIACCiiggAIKVLmAgf4q/wNw9xVQQAEFFFBAAQUUUEABBRRQQAEFFFBAAQUqW8BAf2UfP1uvgAIKKKCAAgoooIACCiiggAIKKKCAAgooUOUCBvqr/A/A3VdAAQUUUEABBRRQQAEFFFBAAQUUUEABBRSobIElK7v5Tdv6KVOmhDFjxoTJkyeHJZdcMqy++uph0KBBYZlllilrQwsXLgwTJ04Mr776avj0009jHausskpYd911Q6dOncqqw4UUUEABBRRQQAEFFFBAAQUUUEABBRRQQAEFFKiPgIH+b7UI0N96663hnXfeCR06dAjz588PTz/9dNhqq63CTjvtFKfVBcu61PHZZ5+Fdu3ahXnz5oURI0aE4cOHh912262sOurahvMVUEABBRRQQAEFFFBAAQUUUEABBRRQQAEFFMgKGOgPIcyZMyfce++9YdSoUWHPPfcMQ4YMCbNmzQoPP/xwuP3220PPnj3DpptumnWr9Zng/s033xzGjx8f9tprr7DmmmuG6dOnh7vvvjv8/e9/D4zsr6uOWpU6QQEFFFBAAQUUUEABBRRQQAEFFFBAAQUUUECBOgQM9IcQPvrooxjo33XXXQMvRuNTCPC//fbb4f777w/rrbdeyRH5zz//fHj55ZfDscceG+8CWHzxbx5/QMoeUvlwd8AGG2wQllpqqToOibMVUEABBRRQQAEFFFBAAQUUUEABBRRQQAEFFChfwIfxhhDGjh0bvvjii7D++uvngvwQdu3aNQwbNizm7f/kk0+KqpKi55lnngl9+/aNdaQgPyusvPLK4ZBDDom5/otW4AwFFFBAAQUUUEABBRRQQAEFFFBAAQUUUEABBRooUPUj+r/++us4op+g/oorrliDkYA9wXserDtp0qT4ucYC336ZOXNmeP/998MWW2wR5s6dGxjd//HHH8eH8a6xxhphm222KfuBvoXqd5oCCiiggAIKKKCAAgoooIACCiiggAIKKKCAAsUEqj7Qv2DBgvD5558HUuy0b9++llPnzp3jg3mnTp1aa16aQD5/7giYMWNG+Mc//hHr42G+THvqqafClltuGR/oS10WBRRQQAEFFFBAAQUUUEABBRRQQAEFFFBAAQWaUqDqA/2M6GcU/mKLLRZf+bgpDc+XX36ZPyv3nc4CXiNHjgz9+/cPu+yyS0zZw50A5PfnYbxdunQJ2267bUj15VbO+zBhwoRw11135U0t/ZV6LQoo0DQCs2fPjhXRgWdRQIGWE/C32HL2brn5BThvpNT3/5pKWY99q6S2Nv8RdwsKNE7A/xMb5+faCjSVgL/FppK0HgX+K/Dwww//90sZn4ib9urVq4wlq3ORqg/0N+Vh54Jm3333jQ/upeNglVVWCd26dYsP9H3ooYfCpptuGu8cKLVNUv7ceuutpRapNW+jjTaqNc0JCijQMAFScVE6dOjQsApcSwEFmkTA32KTMFpJKxVIQXDuBq1PqZT12KdKamt9joHLKtASAv6f2BLqblOB2gL+FmubOEWBxgrUNwZK3NRAf3H1qg/0M8KelD0LFy4sqJSmL7PMMgXnZyeuvvrqoV+/fjXuDOjTp08YMGBAGDVqVEzlQ4qgUoU/1t13373UIrXm1VVnrRWcoIACRQXS6Ep/V0WJnKHAIhHwt7hImN1ICwksueQ3p+D1/b+mUtaDtZLa2kJ/Bm5WgbIF/D+xbCoXVKBZBfwtNiuvlVepQH1joAT6LcUFqj7Qz0UIaXXmzJkTX3zOlunTp4ellloqLpOdnv1MRwEvOgPSRU2az3emz5s3L77S9GLvdAzss88+xWY7XQEFmlkgjdKob/ClmZtl9QpUnYC/xao75FW1w+l8sb7/11TKehzMSmprVf3xubMVKeD/iRV52Gx0GxTwt9gGD6q71OIC9Y2B1jfdeYvv4CJuwOKLeHutbnOM6Ce4/tlnnwVy6mcL+fs/+OCDsMIKK8RXdl72Mw/Z7d27d5g2bVrIz+XPbcv0+pIGpNDDfrP1+FkBBRRQQAEFFFBAAQUUUEABBRRQQAEFFFBAgfoKVH2gH7C11lordOzYMYwePTrMnz8/Z0jw/4UXXgiDBw8OK664Ym56/gdG7A8bNiyMGzcuvPzyy4EOglR4SMSbb74Z1l577bD88sunyb4roIACCiiggAIKKKCAAgoooIACCiiggAIKKNAkAlWfugdFHpq70047hREjRoSuXbvGh+kyCv/RRx8NU6ZMCfvtt1/uwZyM0H/yySfD5MmTww477BBT+iyxxBJhiy22CE899VS4+eab48PHyNc/derU8MADDwRu79puu+1iCp8mOWpWooACCiiggAIKKKCAAgoooIACCiiggAIKKKDAtwIG+kOIQfzddtstPiz3/vvvD88//3xMwUOwf4899oij9RdbbLFIRqD/oYceiiP3N9xww1zu/lVXXTX84Ac/CHfccUd8LbfccjHnP3cI7LvvvmHo0KE1HtLrX6ACCiiggAIKKKCAAgoooIACCiiggAIKKKCAAk0hYKD/W0Vy7B922GFh1KhRYdKkSfEBvGuuuWYYMmRIbjQ/i/JgsW222SYMHDgwF+RnOrn+N9hgg3hHwEsvvRTvBCAvPyl71l13XfPzN8Vfq3UooIACCiiggAIKKKCAAgoooIACCiiggAIK1BIw0P8tCSP2e/T4/+zdCZRcVZ044F+WTmclG9kgEbKAQNj3RSIIAREOCgooCLiA2wgO4zq48R91BkdmBmcUPTCeoygwIouA44KsCiIYHI2yEwIkISGE7Pv6P/dBtd1JV3dVpV91ddX3zmmq6t39u+8l4Ve37xsbaWV/R0cK9B933HHtZknB/ilTpmQ/7WZwkgABAgQIECBAoC4Enl+yNl5YvDYbyy4j+seuw/vXxbgaaRDmsJFm21gJECBAgAABAvUvINBf/3NshAQIECBAgAABAl0kcP+spXHfrCXx/OK12U+qdtcU6B/RP46ZPDzePHlYF7WkmrwEzGFesuolQIAAAQIECBDoTgGB/u7U1zYBAgQIECBAgECPEfh/v5qdBfnvm7W0bZ9nvfbxmMlLsmD/l0+c2Dbdp5oRMIc1MxU6QoAAAQIECBAg0MUCAv1dDKo6AgQIECBAgACB+hO479klcdmdszscWPoCIP2kVf3HTBneYV6J1Rcwh9U31yIBAgQIECBAgED1BHpXryktESBAgAABAgQIEOiZAp0F+VuPqpy8rct5n69AOfNSTt58e612AgQIECBAgAABAqUJCPSX5iQXAQIECBAgQIBAgwo8v3hNpH3dSz1S3lTGUTsC5rB25kJPCBAgQIAAAQIE8hEQ6M/HVa0ECBAgQIAAAQJ1IrDNnvwljKuSMiVUK0uFApXMRyVlKuyeYgQIECBAgAABAgS2W0Cgf7sJVUCAAAECBAgQIFDPAi8sXlv28CopU3YjCpQsUMl8VFKm5A7JSIAAAQIECBAgQKCLBQT6uxhUdQQIECBAgAABAvUlsOuI/mUPqJIyZTeiQMkClcxHJWVK7pCMBAgQIECAAAECBLpYQKC/i0FVR4AAAQIECBAgUF8Cx0wZXvaAKilTdiMKlCxQyXxUUqbkDslIgAABAgQIECBAoIsFBPq7GFR1BAgQIECAAAEC9SWwy/D+8ZYygv0pbyrjqB0Bc1g7c6EnBAgQIECAAAEC+QgI9OfjqlYCBAgQIECAAIE6ErjsxIklj6acvCVXKuN2C5QzL+Xk3e6OqYAAAQIECBAgQIBAFwgI9HcBoioIECBAgAABAgTqW+DoScPiqydN6nBlf1rJn/KkvI7aEzCHtTcnekSAAAECBAgQINB1An27rio1ESBAgAABAgQIEKhfgc8fv2tMmzQsjtt9RMx+dU3898MvxcQR/WPSyAHxlt1GxNEThwry1/j0m8ManyDdI0CAAAECBAgQqFhAoL9iOgUJECBAgAABAgQaTSCtCk8/LyxZG3c/szjbi//LJ0wU4O9BF4I57EGTpasECBAgQIAAAQIlC9i6p2QqGQkQIECAAAECBAi8JpAe7lp44G4KHDt6noA57HlzpscECBAgQIAAAQLFBQT6i9tIIUCAAAECBAgQIECAAAECBAgQIECAAAECNS8g0F/zU6SDBAgQIECAAAECBAgQIECAAAECBAgQIECguIBAf3EbKQQIECBAgAABAgQIECBAgAABAgQIECBAoOYFBPprfop0kAABAgQIECBAgAABAgQIECBAgAABAgQIFBcQ6C9uI4UAAQIECBAgQIAAAQIECBAgQIAAAQIECNS8gEB/zU+RDhIgQIAAAQIECBAgQIAAAQIECBAgQIAAgeICAv3FbaQQIECAAAECBAgQIECAAAECBAgQIECAAIGaFxDor/kp0kECBAgQIECAAAECBAgQIECAAAECBAgQIFBcQKC/uI0UAgQIECBAgAABAgQIECBAgAABAgQIECBQ8wIC/TU/RTpIgAABAgQIECBAgAABAgQIECBAgAABAgSKCwj0F7eRQoAAAQIECBAgQIAAAQIECBAgQIAAAQIEal5AoL/mp0gHCRAgQIAAAQIECBAgQIAAAQIECBAgQIBAcQGB/uI2UggQIECAAAECBAgQIECAAAECBAgQIECAQM0LCPTX/BTpIAECBAgQIECAAAECBAgQIECAAAECBAgQKC4g0F/cRgoBAgQIECBAgAABAgQIECBAgAABAgQIEKh5AYH+mp8iHSRAgAABAgQIECBAgAABAgQIECBAgAABAsUFBPqL20ghQIAAAQIECBAgQIAAAQIECBAgQIAAAQI1LyDQX/NTpIMECBAgQIAAAQIECBAgQIAAAQIECBAgQKC4gEB/cRspBAgQIECAAAECBAgQIECAAAECBAgQIECg5gUE+mt+inSQAAECBAgQIECAAAECBAgQIECAAAECBAgUFxDoL24jhQABAgQIECBAgAABAgQIECBAgAABAgQI1LyAQH/NT5EOEiBAgAABAgQIECBAgAABAgQIECBAgACB4gIC/cVtpBAgQIAAAQIECBAgQIAAAQIECBAgQIAAgZoXEOiv+SnSQQIECBAgQIAAAQIECBAgQIAAAQIECBAgUFxAoL+4jRQCBAgQIECAAAECBAgQIECAAAECBAgQIFDzAgL9NT9FOkiAAAECBAgQIECAAAECBAgQIECAAAECBIoLCPQXt5FCgAABAgQIECBAgAABAgQIECBAgAABAgRqXkCgv+anSAcJECBAgAABAgQIECBAgAABAgQIECBAgEBxAYH+4jZSCBAgQIAAAQIECBAgQIAAAQIECBAgQIBAzQsI9Nf8FOkgAQIECBAgQIAAAQIECBAgQIAAAQIECBAoLiDQX9xGCgECBAgQIECAAAECBAgQIECAAAECBAgQqHkBgf6anyIdJECAAAECBAgQIECAAAECBAgQIECAAAECxQUE+ovbSCFAgAABAgQIECBAgAABAgQIECBAgAABAjUvINBf81OkgwQIECBAgAABAgQIECBAgAABAgQIECBAoLiAQH9xGykECBAgQIAAAQIECBAgQIAAAQIECBAgQKDmBQT6a36KdJAAAQIECBAgQIAAAQIECBAgQIAAAQIECBQXEOgvbiOFAAECBAgQIECAAAECBAgQIECAAAECBAjUvIBAf81PkQ4SIECAAAECBAgQIECAAAECBAgQIECAAIHiAgL9xW2kECBAgAABAgQIECBAgAABAgQIECBAgACBmhcQ6K/5KdJBAgQIECBAgAABAgQIECBAgAABAgQIECBQXECgv7iNFAIECBAgQIAAAQIECBAgQIAAAQIECBAgUPMCAv01P0U6SIAAAQIECBAgQIAAAQIECBAgQIAAAQIEigsI9Be3kUKAAAECBAgQIECAAAECBAgQIECAAAECBGpeQKC/5qdIBwkQIECAAAECBAgQIECAAAECBAgQIECAQHEBgf7iNlIIECBAgAABAgQIECBAgAABAgQIECBAgEDNCwj01/wU6SABAgQIECBAgAABAgQIECBAgAABAgQIECgu0Ld4UmOlbNmyJV566aW455574vnnn49+/frFfvvtF0cddVQMGTKkJIyHH344fvnLX7ab98gjj4zp06e3m+YkAQIECBAgQIAAAQIECBAgQIAAAQIECBCoVECg/3W52bNnxzXXXBOrV6+OXXbZJXu9+eabs6D/Oeec02mwf9OmTfHoo4/GAw88EFOnTo2mpqY2c7Jy5co2n30gQIAAAQIECBAgQIAAAQIECBAgQIAAAQJdISDQHxErVqyIFNR/9dVX4/zzz4/JkyfHunXr4qGHHopbb701Jk2alK3G79WrV1HzDRs2xPz582PvvfeOD37wg9Hc3Nwmb6m/FdCmkA8ECBAgQIAAAQIECBAgQIAAAQIECBAgQKATAYH+iGzV/r333psF+Q877LDo2/c1lmHDhmWr9O+66644/PDDY4cddijKmVbsz507Nw4++ODYa6+9ondvjz8oiiWBAAECBAgQIECAAAECBAgQIECAAAECBLpMQDQ6Ip5++ulIK/LTljuFIH8SToH9tE//Y489FgsXLuwQffHixdmK/vHjxwvydyglkQABAgQIECBAgAABAgQIECBAgAABAgS6UqDhV/SnvfXTQ3hHjhwZO+64YxvbtFXPhAkTYsmSJbFgwYKYMmVKm/TWH9IXAatWrYpU3//8z//EM888k+3Tn1b3T5s2LdJvBzgIECBAgAABAgRqS+CJhaviyZdXZ53aY8zA2HP0oNrqoN7UnIBrpuamRIcIECBAgAABAgQiouED/Rs3bswC+f37999mX/10hQwaNChSnmXLlnV4wbzwwgvZbwWkbX523333GDFiRPYFQtr7/4knnoj3v//9MXr06A7rkEiAAAECBAgQIFAdgVv/8krc8pdX4uUV67Of1OqYIf2yn9P3GRWn7TOqOh3RSo8RcM30mKnSUQIECBAgQIBAQwo0fKB/y5YtsX79+ij2oN3C+RTsL3aktPQg3xTcT6v3DznkkBg6dGj2BcL999+fPeg3pZ133nntfplQrF7nCRAgQIAAAQIEul7g3Osfj5kvrYyZ81e2rXz+ax9TWvoS4Idn79U23aeGFXDNNOzUGzgBAgQIECBAoMcINHygvytmKj1494QTTogDDzww9t9//xg8eHBWbdoKKAX8H3/88UgP+z355JNjp5126rDJWbNmxXe+850O82ydePbZZ299ymcCBCoUWL58eVZywIABFdagGAECXSHgXuwKRXW0J/CLZ1fFjx5d0F5Sy7n0BUD6OfmNw+KkKcW38knPeEpHZ7/52VLx62/qvVwaZj2NsSuvma2vBZ8JlCLg78RSlOQhkL+AezF/Yy00nsD1119f1qBT3HTy5MlllWmkzA0f6O/Tp08MHDgw0sr+9LP1sXnz5uxUIXi/dXr6nAL9e+yxR3tJMWrUqDjggAPi6quvzlb9dxboT78Z8Lvf/a7duoqdPO2004olOU+AQJkCa9euzUqsWbOmzJKyEyDQlQLuxa7UVFdrga/c9ULrjx2+T3mP2Xli0Tzp2UzpKPfvjHovl0zqaYxdec0UvZgkEOhAwN+JHeBIIlBFAfdiFbE11TAC5cZAU9xUoL/45dHwgf6+fftmW+6sXLkyVq9enb1vzZUexNuvX79OH6abviRIK5dS3tZH2von7f+ftvfpaPufQpl0sX7sYx8rfCzp1YN+S2KSiUBJAoV/vLmvSuKSiUBuAu7F3GgbuuLHX14Vjy8s/YvclPeldU2x15j2V/U3NTVlnuX+nVHv5RJKvYyxq6+Zhr4BDb5iAX8nVkynIIEuFXAvdimnyghkAuXGQFOg31FcoOED/SkQv+uuu2ar7RcsWBDjx49v0UorkWbPnh1jx47t8EG66cuAH/zgB1meM844o+V/bFJFKfg/d+7cGDNmTIwcObKl7mJvUp4jjjiiWLLzBAjkLJC+mEtH4TXn5lRPgEARgcI9WHgtks1pAmUJPDyv/P8xeHje2jhwl/b/DZd+MzQd5V6n9V4umdTLGLv6minrgpWZwOsChT9jCq9gCBDoHoHCPVh47Z5eaJVAfQmUGwMtJbZaX0LljaZ3ednrM/fuu++eBel///vfx7p161oGmQL/6dxBBx2UbcHTkrDVm/Q/MnPmzInbbrst+2KgdXLaO6pQR9qz30GAAAECBAgQINA9AgtXrC+74UrKlN2IAjUrUMn8V1KmZgF0jAABAgQIECBAoMcINPyK/jRTO++8c6R97m+55ZZobm6Oo446KtJDVv73f/83W5F/0kkntazUSiv0U74XX3wxzj///OwLgrR/f3rQ7pVXXhnXXHNN9j7VmYL8d9xxR7adT0r3cM8ec1/oKAECBAgQIFCHAnuOHVj2qCopU3YjCtSsQCXzX0mZmgXQMQIECBAgQIAAgR4jINAfkQX3p0+fHmm/tUceeSQeffTR7MG8ab/VD3zgA7Hnnnu2TGjazucvf/lLzJw5M975zndmgf70MN5DDz0021v/wQcfjBtuuCHSlkDpJ2378973vjemTp2afW6pyBsCBAgQIECAAIGqChwzeXjZ7VVSpuxGFKhZgUrmv5IyNQugYwQIECBAgAABAj1GQKD/9alKezy94x3viH322SeWLl2a7Ss6bty4bP/+tMq/cKQHi5111lnx1re+NQvyF86nVf3HHnts7LbbbjFv3rxsC6BBgwZlvy2Q9v0v7FNayO+VAAECBAgQIECgugKjB/eLd+07Om6aubCkhlPeVMbRuAKumcadeyMnQIAAAQIECPQ0AYH+VjOWVvCn/fg7OlLAPn0Z0N6RvhCYPHly9tNeunMECBAgQIAAAQLdK3DZiRNLDvSnvA4CrhnXAAECBAgQIECAQE8Q8DDenjBL+kiAAAECBAgQINAlAlPHDoofn7d3nLHv6KL1pbSUJ+V1EHDNuAYIECBAgAABAgR6goAV/T1hlvSRAAECBAgQIECgywTO3G90TB0zKN61/+h4fMGq+H93zo69xgxqOZfSBPm7jLsuKnLN1MU0GgQBAgQIECBAoK4FBPrrenoNjgABAgQIECBAoD2BFMhPP69MXh83/XlhjBrcFGmLlr2s4m+Py7mI7HpxzbgUCBAgQIAAAQIEalXA1j21OjP6RYAAAQIECBAgkLvAqMH9siB/akiQP3fuumjANVMX02gQBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaFiLmwEAACAASURBVCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCQBgf5Gmm1jJUCAAAECBAgQIECAAAECBAgQIECAAIG6ExDor7spNSACBAgQIECAAAECBAgQIECAAAECBAgQaCSBvo00WGMlQIAAAQIECBCoP4FH566IP85dkQ3swPFD4qDxQ+pvkEbU4wVcpz1+Cg2AAAECBAgQIFDTAgL9r0/Pli1bYvbs2XHTTTfFk08+Gc3NzXHUUUfFKaecEsOGDSt7ElN9999/f9x+++3xyU9+Mnbeeeey61CAAAECBAgQIECguMA1D70UVz88L1as2xQr1m7KMg7p3yeGNPeJDx22c1x4xE7FC0shUCUB12mVoDVDgAABAgQIEGhwAYH+1y+Axx9/PK644ooYOnRoHHfccbFs2bL4xS9+EbNmzYqLL744hg8fXtalMm/evLj22mtj0aJFsW7durLKykyAAAECBAgQINCxwCFX/iFeWrY+Xlq+1b+zlr9W7qVls7MvAf7w94d0XJFUAjkKuE5zxFU1AQIECBAgQIBAGwF79EfE0qVL47rrrsuC/H/3d38Xp556arznPe+JD33oQzFz5sy45557Iq3QL/VYs2ZN/OQnP4mnnnqq1CLyESBAgAABAgQIlChw9UMvxYw5K7YN8rcqn74ASHlSXgeB7hBwnXaHujYJECBAgAABAo0rINAfka3af/jhh2P69OkxefLkGDJkSLaC/5BDDol99tknC/SnLwNKOdIXAqmuRx55JI488shSishDgAABAgQIECBQhsBld84uOXc5eUuuVEYCJQiUc+2Vk7eEpmUhQIAAAQIECBBoQAGB/ohs5X1TU1NMmTIlevf+G8mAAQNir732imeeeSZefvnlki6PtGXPDTfcEMcff3wcdNBBJZWRiQABAgQIECBAoDSBGXOXx/ytt+vpoGjKm8o4CFRTwHVaTW1tESBAgAABAgQIJIG/RbUb1GPTpk2xcOHCGDlyZIwYMaKNQq9evbKH6C5fvjwWLFjQJq29D+vXr49bb701+vfvH29729sifVHgIECAAAECBAgQ6DqB+58t7bcsW7dYSZnW5b0nUK5AJddcJWXK7Zf8BAgQIECAAAEC9SvQ8IH+jRs3xpIlS6Jfv37Rt++2zyZubm6OzZs3x4oVKzq9CtKWPb/5zW/izDPPjDFjxnSaXwYCBAgQIECAAIHyBFau21RegYiopEzZjShAoJVAJddcJWVaNektAQIECBAgQIBAgwtsG9luMJC0p35a1V/sSKv605GC/R0dacV/eqBv2pc/bdnTegugjsptnfbXv/41vvCFL2x9usPPf//3f99hukQCBEoXWLx4cZa5vS/+Sq9FTgIEtlfAvbi9gvVbfvdh5Y8tlVm0aFHRguvWrcvSOsrTXmHl2lN57Vyj2+RxnRbXllLvAv5OrPcZNr6eIuBe7CkzpZ89SeDKK68sq7spbrr33nuXVaaRMjd8oL8rJjtt2fPTn/40q+r000/Ptu6ptN7Vq1fHCy+8UFbx9FsJDgIEukagcD8VXrumVrUQIFCuQOEeLLyWW17++hU4dKfmsgeXynR0LaWFH+noKE97jSrXnspr5xrdJo/rtLi2lHoXKPzZVHit9/EaH4FaFSjcg4XXWu2nfhHoSQLlxkBT3NRRXKDhA/19+vSJIUOGZCv2C/9D0pqrsJJ/6NChrU+3ef/oo4/GnXfeGR/96Edj/PjxbdLK/ZC+lfra175WVrEdd9yxrPwyEyBQXKDwjzb3VXEjKQSqIeBerIZyz2wj/avnI0esiO8+NK+kAXzkiJ1jl3GjO8zb3PxaXeX+2a9ccdZGt8njOi2uLaXeBfydWO8zbHw9RcC92FNmSj97kkC5MdBLLrmkJw2v6n1t+EB/2p4jPYg3PXA37cO/9QN5X3nllWyF/rBh7f+eePq15Pvvvz/mz58f3/nOd+L73/9+yySmOl999dX47Gc/GzvttFN8+tOf7vSLgIEDB8Yb3vCGljq8IUCgugKFLXsKr9VtXWsECBQECvdg4bVw3iuBJHDZiRNLDvSnvJ1dR4WtGjvLt7W+cluL/O0zm66/Tv+m612jCRT+bCq8Ntr4jZdArQgU7sHCa630Sz8I9GSBcmOgKW7qKC7Q8IH+9D8hkyZNygLyKVi/yy67tGilvftnzZoVO++8c9GH66a9+I8++uh44xvf2FKu8GbmzJnZw3lPOOGErI7BgwcXkrwSIECAAAECBAhUKDBmSL/486cOje8+OC++U2Rl/0eP2Dk+ctTOkfI6CHSHgOu0O9S1SYAAAQIECBBoXIGGD/SnqU9B+l133TXuu+++2G+//WLAgAHZFZH2ifrtb38bb3rTm6LYr3Knb3LTw3cLW/y0vpTSr3XNmDEjjjrqqJg4cWI0N5e/p2zr+rwnQIAAAQIECBB4TWDfcYPjyydOzIL5M+Ysjw/++Mk4eMKQOGT8Dq8F+Af3E+R3sXS7gOu026dABwgQIECAAAECDSMg0B+RrdY/++yz49vf/nb2ELYTTzwxlixZEjfddFMMGjQoTj755GhqasouirRVz9VXXx1PP/10tiVP2pO/f//+7V4w/fr1i/QbAym98OVBuxmdJECAAAECBAgQKFsgrZhOP5NHDojv/m5eDOrXJy5768QYPdgq/rIxFchNwHWaG62KCRAgQIAAAQIEWgkI9Edk+7ZOmzYt0sN477jjjviXf/mXSA/p3WeffeL0009vs51PyvPyyy9HWu2/fv36VpTeEiBAgAABAgQIdIdACvCnn3QI8nfHDGizFAHXaSlK8hAgQIAAAQIECFQqIND/ulx6mMOxxx6bbd2zdu3aSHvvpz31hw8fngX9C8Bp+52LLrooUp5x48YVTrf7murbf//9O83XbmEnCRAgQIAAAQIECBAgQIAAAQIECBAgQIBACQIC/a2Q0hY7O+20U6sz275NW/GMGTNm24R2zuywww6RfhwECBAgQIAAAQIECBAgQIAAAQIECBAgQCAvgd55VaxeAgQIECBAgAABAgQIECBAgAABAgQIECBAIH8Bgf78jbVAgAABAgQIECBAgAABAgQIECBAgAABAgRyExDoz41WxQQIECBAgAABAgQIECBAgAABAgQIECBAIH8Bgf78jbVAgAABAgQIECBAgAABAgQIECBAgAABAgRyExDoz41WxQQIECBAgAABAgQIECBAgAABAgQIECBAIH8Bgf78jbVAgAABAgQIECBAgAABAgQIECBAgAABAgRyExDoz41WxQQIECBAgAABAgQIECBAgAABAgQIECBAIH8Bgf78jbVAgAABAgQIECBAgAABAgQIECBAgAABAgRyExDoz41WxQQIECBAgAABAgQIECBAgAABAgQIECBAIH8Bgf78jbVAgAABAgQIECBAgAABAgQIECBAgAABAgRyExDoz41WxQQIECBAgAABAgQIECBAgAABAgQIECBAIH8Bgf78jbVAgAABAgQIECBAgAABAgQIECBAgAABAgRyExDoz41WxQQIECBAgAABAgQIECBAgAABAgQIECBAIH8Bgf78jbVAgAABAgQIECBAgAABAgQIECBAgAABAgRyExDoz41WxQQIECBAgAABAgQIECBAgAABAgQIECBAIH8Bgf78jbVAgAABAgQIECBAgAABAgQIECBAgAABAgRyExDoz41WxQQIECBAgAABAgQIECBAgAABAgQIECBAIH8Bgf78jbVAgAABAgQIECBAgAABAgQIECBAgAABAgRyExDoz41WxQQIECBAgAABAgQIECBAgAABAgQIECBAIH8Bgf78jbVAgAABAgQIECBAgAABAgQIECBAgAABAgRyExDoz41WxQQIECBAgAABAgQIECBAgAABAgQIECBAIH8Bgf78jbVAgAABAgQIECBAgAABAgQIECBAgAABAgRyExDoz41WxQQIECBAgAABAgQIECBAgAABAgQIECBAIH8Bgf78jbVAgAABAgQIECBAgAABAgQIECBAgAABAgRyExDoz41WxQQIECBAgAABAgQIECBAgAABAgQIECBAIH8Bgf78jbVAgAABAgQIECBAgAABAgQIECBAgAABAgRyExDoz41WxQQIECBAgAABAgQIECBAgAABAgQIECBAIH8Bgf78jbVAgAABAgQIECBAgAABAgQIECBAgAABAgRyExDoz41WxQQIECBAgAABAgQIECBAgAABAgQIECBAIH8Bgf78jbVAgAABAgQIECBAgAABAgQIECBAgAABAgRyExDoz41WxQQIECBAgAABAgQIECBAgAABAgQIECBAIH8Bgf78jbVAgAABAgQIECBAgAABAgQIECBAgAABAgRyExDoz41WxQQIECBAgAABAgQIECBAgAABAgQIECBAIH8Bgf78jbVAgAABAgQIECBAgAABAgQIECBAgAABAgRyExDoz41WxQQIECBAgAABAgQIECBAgAABAgQIECBAIH8Bgf78jbVAgAABAgQIECBAgAABAgQIECBAgAABAgRyExDoz41WxQQIECBAgAABAgQIECBAgAABAgQIECBAIH8Bgf78jbVAgAABAgQIECBAgAABAgQIECBAgAABAgRyExDoz41WxQQIECBAgAABAgQIECBAgAABAgQIECBAIH8Bgf78jbVAgAABAgQIECBAgAABAgQIECBAgAABAgRyE6h6oH/VqlXxf//3f5FeHQQIECBAgAABAgQIECBAgAABAgQIECBAgMD2CVQ90L9gwYL4t3/7t0ivHR233XZbTJkypdOfL37xi7Fu3bqOqpJGgAABAgQIECBAgAABAgQIECBAgAABAgTqVqBvtUe2efPmWL16daTXGTNmxFlnnbVNF/793/89C94fcMABcc4550S/fv3i9ttvz/KdeuqpsX79+rjuuutir732ilNOOSX69q36MLbpsxMECBAgQIAAAQIECBAgQIAAAQIECBAgQKA7BLo1Qr5x48aYMGFCfPzjH49BgwZFWsWfjn333Tf+8Ic/xE477RTHH3989O/fP2bOnJmlnXDCCbF27dq49957Y4899ogDDzww+vTp0x122iRAgAABAgQIECBAgAABAgQIECBAgAABAt0u0K2B/jT6ESNGxLRp07LXP//5zxnILrvskgX6e/funa3WTyv20/t0pPeFzynAL8ifsfgPAQIECBAgQIAAAQIECBAgQIAAAQIECDSoQLcH+nv16tUmeJ/moRDUb9A5MWwCBAgQIECAAAECBAgQIECAAAECBAgQIFCyQLcH+jvq6f333x8XXHBBFvifNWtWlvWvf/1rtr//448/HkcccURHxaURIECAAAECBAgQIECAAAECBAgQIECAAIG6F6jpQP+OO+4YU6dOjaampuwBvmk20v79GzZsiIULF9b95BggAQIECBAgQIAAAQIECBAgQIAAAQIECBDoTKCmA/3pYbsf+9jHsofxpuB+Oi666KLsYbzz58/vbGzSCRAgQIAAAQIECBAgQIAAAQIECBAgQIBA3QvUdKA/PWi3ubk5+yk8dDd93rJli4fw1v2laYAECBAgQIAAAQIECBAgQIAAAQIECBAgUIpAtwf6//znP8dHP/rR6NevXxT24T/ggANK6bs8BAgQIECAAAECBAgQIECAAAECBAgQIECg4QW6NdCf9uC/8MILW1bnp/330zF06NBYsmRJ3HffffGBD3wgexjvc889l6XNnDkzexjvE088EX/5y1/ixRdfjI9//OPZ9j4NP5sACBAgQIAAAQIECBAgQIAAAQIECBAgQKDhBLo10D9p0qS45JJLtkFP2/Sk7XnOPvvs6NWrV/az3377Zfl69+6dpaVV/yntDW94Q/ZFwDaVOEGAAAECBAgQIECAAAECBAgQIECAAAECBBpAoOqB/l122SW+/e1vx8iRI2P9+vWxZs2adpknT54cu+66a5u0QYMGZVv8tD6Zgv1NTU2tT3lPgAABAgQIECBAgAABAgQIECBAgAABAgQaRqDqgf60F//YsWOz1fi33HJLXHTRRSVjH3jggfGf//mfMXHixJLLyEiAAAECBAgQIECAAAECBAgQIECAAAECBOpZoOqB/oSZVuGnY+PGjXHyySfHZz7zmWhubu7QedGiRXHppZfGwoULBfo7lJJIgAABAgQIECBAgAABAgQIECBAgAABAo0kUNVA/6pVq+Lxxx/PAvwJ+fnnn4/Vq1fH3Llzt9mSZ+tJWLlyZZx00kmR6njooYdaknfYYYfYY489Wh7o25LgDQECBAgQIECAAAECBAgQIECAAAECBAgQaACBqgb658+fH5/73Ofisccey2g3b94c6eeuu+4qiTrlTQ/jbX2cd9558U//9E8C/a1RvCdAgAABAgQI9ECB385eGg/MWpb1/E2Th8bRE4f1wFHoMoGuF3BvdL2pGgkQIECAAAEC9SZQ1UB/ehDvdddd17Ki/xe/+EXMmDEjPvvZz3a6or8YfP/+/Tvd9qdYWecJECBAgAABAgS6X+Cf73o+/vnuF2LTli2xafOWrEN9eveKPr16xaXH7RKXHr9r93dSDwh0g4B7oxvQNUmAAAECBAgQ6KECVQ30NzU1xZgxY1qohg4dGnPmzInbbrstUlopxxvf+MY4/vjj22Qt7Pnf5qQPBAgQIECAAAECNS+QApmX3Tk7Nmx6LcBf6HDhc0pLh2B/QcZrowi4Nxplpo2TAAECBAgQINA1AlUN9Kcutw7KDx48ONKK/AcffLDk0fTr169NHSUXlJEAAQIECBAgQKCmBIoFMlt3MgX8Bftbi3jfCALujUaYZWMkQIAAAQIECHStQNUD/a27f+KJJ8Zxxx3X+lSn7/v06dNpnkoybNmyJZ588sm45ppr4k9/+lP2BUTq33vf+94YOXJkSVW++uqrceutt8bPfvazWLx4cey2227x7ne/O6ZNm2Z7oZIEZSJAgAABAgQaSaC9lfztjb8Q7Leqvz0d5+pRwL1Rj7NqTAQIECBAgACBfAXaPtk237a2qT0F7Zubm9v8pBX769aty/bx3zotfe7b97XvJl5++eXsob4pQN8Vx6OPPhqXXHJJpPr+8R//Mc4999y4995740tf+lIsWrSo0yZSnpT3pz/9aZxyyinx1a9+NaZOnRpf+cpX4kc/+lHLcwk6rUgGAgQIECBAgEADCPzmuaXbbNfT0bBTsD+VcRCodwH3Rr3PsPERIECAAAECBPIRqHqgf+3atfHpT386br755mxEKbC+fPnyWLVqVfY5BflTcPy73/1uzJ07Nwuev/LKK21Gv3Hjxuyhvmm1/B/+8Ic2aZV8SCvxv/e978UBBxwQl112WfZbBmeeeWZ8/etfz/pwxx13ZF8AFKs7jSHleeqpp+LSSy+N97///fGmN70pLr744jjvvPPi+uuvz76UKFbeeQIECBAgQIBAownc/+ySsodcSZmyG1GAQDcLVHKdV1Kmm4epeQIECBAgQIAAgS4WqHqgP/U/Beo3bdqUDaV1YL8wtkL6iy++mD2oN22nk84VjhTcv/baa2P69OnZqvnC+UpfU4B+5syZ8da3vjXSA4J79+4d6bcNJk2aFEcddVT85je/ifRlQLFjzZo1kX7D4OCDD4599903K5vqSL99sP/++8eKFSvihRdeKFbceQIECBAgQIBAwwlsruCXMisp03CwBtzjBSq5zisp0+OhDIAAAQIECBAgQKCNQLcE+tv0YKvAf+u0ww47LNtGJwX1H3jggSzp97//ffzDP/xD7LnnntlvBgwaNKh1kYreP/300zFs2LAYP358m/JNTU2x++67x/PPPx8LFixok9b6w8CBA7O+pO160gOGWx9Llry2Wm348OGtT3tPgAABAgQIEGhogaMnDyt7/JWUKbsRBQh0s0Al13klZbp5mJonQIAAAQIECBDoYoFufRhvZ2NJq+pPP/30mDdvXlxxxRXx17/+NdtiZ9ddd8222Bk7dmxnVXSann5TIK3WTw/cbS8YP27cuGxbobRif++99y5a39YPCV6/fn088sgjcdVVV8URRxzRYdmilUogQIAAAQIECNSpwDGTy18EUUmZOuUzrDoWqOQ6r6RMHRMaGgECBAgQIECgIQVqYkV/R/JpC5y0qj6tuv/yl78cxx57bFx99dXZuV69enVUtKS0FOhPq+5ToL69+goP/125cmVJ9aVMN954Y9bP9CyC9EVB2re/vS8RSq5QRgIECBAgQIBAnQn07hXxxeN3LXlUKW8q4yBQ7wLujXqfYeMjQIAAAQIECOQjULOB/tmzZ8fll18e06ZNyx5ue8ghh2T79X/qU5/KAul33nlnPiJdUGva2/8b3/hGXHDBBdm2P5dcckk899xzXVCzKggQIECAAAEC9SPw5RMnljyYcvKWXKmMBGpUoJzrvZy8NTpc3SJAgAABAgQIEOgCgZrduufHP/5xHHTQQXHGGWfECSeckK3gT6vrN2/eHMuWLcsexnvggQfGqFGjuoCha6tI/U7H4YcfHukLis997nPZbyF88YtfjM6eKZAeNPy+972vrA6lL0QcBAh0jcDChQu7piK1ECCwXQLuxe3iq8vCryx8ucNxrV27Nkvv6LlKxSqotKxyxUQj2LRvU6lL+7W9draze6OjstJ6hoC/E3vGPOll/Qu4F+t/jo2w+gIpZlrOkeKmKdbqaF+gZgP9n/nMZyKt3k9b96QtdQrb6qQtdt7xjnfE7bffHvfdd1/2RUD7QyvtbPryID2Id86cObFly5ZtCm3atCk7N2LEiG3Sip0o9DW9vvGNb4wjjzwy0oWY9vlPq/07OlIfCm12lE8aAQIECBAgQKAeBF785J4tw3hk3pp41/88H4ft1D+mTR4eFx1a/gN7WyrzhkAPF3Bv9PAJ1H0CBAgQIECgU4FyY6DtxW47baSBMtRsoH/+/Pnx8MMPtzsVaV/9qVOnxs0335xt7TNmzJh285VyMgX604N40z79S5cuja0D+qkfAwcOzL4MKKW+rfM0NzfH0KFD2/0SYeu86XP6Vur73/9+e0lFz239IOCiGSUQIFCyQFc87LvkxmQkQKCogHuxKE1dJowftyXe/MjSSI9huvzte0TvMp7H1L//S5lJJddMpWWVK34ZsmnfplKX7bk32u+Jsz1RoJI/33riOPWZQK0LuBdrfYb0rycJlBsDPf/883vS8Kre15oN9F977bXxox/9qChI2sInHSnY/5GPfCRb+V80cycJu+22WyxatCjmzZvXZsV9+kLh2WefjV122SU6+oN87ty52fME0mr9iy++OAoP8E3Npl/PTSv5hwwZkv100pXsNxcE7jtTkk6AAAECBAjUo8Brv8X52sjKCfLXo4UxEWgt4N5oreE9AQIECBAgUC8C5cZAC7uo1Mv4u3oc3R7ob2pqyh62269fvzZjSw+w/ehHP9rmXOsPaf/V9EDe008/fbuC/KnOPfbYI/sNgV/96leR9v0v7KOfgvz33HNPnHbaabHjjju2br7N+/ScgNGjR8f9998fb3/722Py5Mkt6X/605/iwQcfjDPPPDOGDx/ect4bAgQIECBAgAABAgQIECBAgAABAgQIECDQFQLdFuh/7rnn4oEHHmgzhvR5/fr1kR5wklbsz5o1K1th379//2w1fOtvbVJwfe+9927Zu79NRWV+SEH8Cy64IC699NLswbnvfOc7sxX+P/zhD2PnnXeOU089taWdtEL/a1/7WsycOTP+4z/+I+tf2p7nnHPOyc5ddNFFWVA/7c0/Y8aMSA8V3n///SPV2Xqlf5ldlJ0AAQIECBAgQIAAAQIECBAgQIAAAQIECLQr0G2B/q9//euRfjo60vY9heB+Wg1/6KGHxvTp0+PEE0+MnXbaqSWtozpKTUt1X3HFFXH11VfH5z//+RgwYECcdNJJcd5550X6UqH1kR78sPXDH9Iq/m9961vZdkM33nhjvPrqqzFhwoRsW6H0GwGF3xJoXY/3BAgQIECAAAECBAgQIECAAAECBAgQIEBgewWqHuhPq98vu+yy+MIXvtBp39Oe+cuWLYvZs2dnq+UfeeSR+OQnP5n9vPWtb42PfexjcdRRR2331j2pI+kLhQMOOCCuuuqqln4VvmRoORER6bcLvvKVr2Sntk5P+/h/6lOfyvpXKLN1nsJ5rwQIECBAgAABAgQIECBAgAABAgQIECBAoCsEqh7oT4HvoUOHZn1fvHhxfOhDH8qC44cffnh2Lm2N86//+q/ZivnPfe5zkb4YOProo7PPaRV92tbntttuix/84AfZljrf/OY3s1X3XYGR6iglMN9Zns7Su6qv6iFAXnmDAgAAIABJREFUgAABAgQIECBAgAABAgQIECBAgAABAr27m2DrLXDSivn0cNyf//zn2Ur+Qv9S8Lx3796RVs1/+MMfjrvuuivuuOOOOOusswpZvBIgQIAAAQIECBAgQIAAAQIECBAgQIAAgYYTqOqK/lWrVmUPp12+fHkGnVbvz507N2655Zb4/e9/34K/aNGiSKv9/+u//it22223lvNbv0kPvHUQIECAAAECBAgQIECAAAECBAgQIECAAIFGFqhqoH/dunXZSv277767xTyt6H/66adbPrd+c91113W4lU4qm74MOPfcc1sX854AAQIECBAgQIAAAQIECBAgQIAAAQIECDSMQFW37hkxYkTcdNNNsWTJkuxn1qxZcdxxx8UvfvGLlnOFtOuvvz7bpufee+/dJi3leeqpp+LNb35zzJ8/v2Emy0AJECBAgAABAgQIECBAgAABAgQIECBAgMDWAlUN9G/dePpc7MG1e+21V0yYMCEefPDB7EG8W5ft27dvNDU1bX3aZwIECBAgQIAAAQIECBAgQIAAAQIECBAg0FACVd26Z2vZ5ubmOPnkk2PUqFFbJ2Wr+S+88MLYvHlzpC1/0kN6Wx8pyD99+vTswb2tz3tPgAABAgQIECBAgAABAgQIECBAgAABAgQaSaBbA/2DBg2K97///e16py8BzjzzzHbT0skhQ4bEhz/84aLpEggQIECAAAECBAgQIECAAAECBAgQIECAQCMIdPvWPY2AbIwECBAgQIAAAQIECBAgQIAAAQIECBAgQCAvAYH+vGTVS4AAAQIECBAgQIAAAQIECBAgQIAAAQIEqiAg0F8FZE0QIECAAAECBAgQIECAAAECBAgQIECAAIG8BAT685JVLwECBAgQIECAAAECBAgQIECAAAECBAgQqIKAQH8VkDVBgAABAgQIECBAgAABAgQIECBAgAABAgTyEhDoz0tWvQQIECBAgAABAgQIECBAgAABAgQIECBAoAoCAv1VQNYEAQIECBAgQIAAAQIECBAgQIAAAQIECBDIS0CgPy9Z9RIgQIAAAQIECBAgQIAAAQIECBAgQIAAgSoICPRXAVkTBAgQIECAAAECBAgQIECAAAECBAgQIEAgLwGB/rxk1UuAAAECBAgQIECAAAECBAgQIECAAAECBKogINBfBWRNECBAgAABAgQIECBAgAABAgQIECBAgACBvAQE+vOSVS8BAgQIECBAgAABAgQIECBAgAABAgQIEKiCgEB/FZA1QYAAAQIECBAgQIAAAQIECBAgQIAAAQIE8hIQ6M9LVr0ECBAgQIAAAQIECBAgQIAAAQIECBAgQKAKAgL9VUDWBAECBAgQIECAAAECBAgQIECAAAECBAgQyEtAoD8vWfUSIECAAAECBAgQIECAAAECBAgQIECAAIEqCAj0VwFZEwQIECBAgAABAgQIECBAgAABAgQIECBAIC8Bgf68ZNVLgAABAgQIECBAgAABAgQIECBAgAABAgSqICDQXwVkTRAgQIAAAQIECBAgQIAAAQIECBAgQIAAgbwEBPrzklUvAQIECBAgQIAAAQIECBAgQIAAAQIECBCogoBAfxWQNUGAAAECBAgQIECAAAECBAgQIECAAAECBPISEOjPS1a9BAgQIECAAAECBAgQIECAAAECBAgQIECgCgIC/VVA1gQBAgQIECBAgAABAgQIECBAgAABAgQIEMhLQKA/L1n1EiBAgAABAgQIECBAgAABAgQIECBAgACBKggI9FcBWRMECBAgQIAAAQIECBAgQIAAAQIECBAgQCAvAYH+vGTVS4AAAQIECBAgQIAAAQIECBAgQIAAAQIEqiAg0F8FZE0QIECAAAECBAgQIECAAAECBAgQIECAAIG8BAT685JVLwECBAgQIECAAAECBAgQIECAAAECBAgQqIKAQH8VkDVBgAABAgQIECBAgAABAgQIECBAgAABAgTyEhDoz0tWvQQIECBAgAABAgQIECBAgAABAgQIECBAoAoCAv1VQNYEAQIECBAgQIAAAQIECBAgQIAAAQIECBDIS0CgPy9Z9RIgQIAAAQIECBAgQIAAAQIECBAgQIAAgSoICPRXAVkTBAgQIECAAAECBAgQIECAAAECBAgQIEAgLwGB/rxk1UuAAAECBAgQIECAAAECBAgQIECAAAECBKogINBfBWRNECBAgAABAgQIECBAgAABAgQIECBAgACBvAQE+vOSVS8BAgQIECBAgAABAgQIECBAgAABAgQIEKiCgEB/FZA1QYAAAQIECBAgQIAAAQIECBAgQIAAAQIE8hIQ6M9LVr0ECBAgQIAAAQIECBAgQIAAAQIECBAgQKAKAgL9VUDWBAECBAgQIECAAAECBAgQIECAAAECBAgQyEtAoD8vWfUSIECAAAECBAgQIECAAAECBAgQIECAAIEqCAj0VwFZEwQIECBAgAABAgQIECBAgAABAgQIECBAIC8Bgf68ZNVLgAABAgQIECBAgAABAgQIECBAgAABAgSqICDQXwVkTRAgQIAAAQIECBAgQIAAAQIECBAgQIAAgbwEBPrzklUvAQIECBAgQIAAAQIECBAgQIAAAQIECBCogoBAfxWQNUGAAAECBAgQIECAAAECBAgQIECAAAECBPISEOjPS1a9BAgQIECAAAECBAgQIECAAAECBAgQIECgCgJ9q9CGJggQIECAAAECBBpIYN3GzZF+0tHct3f200DDN1QCNSPgXqyZqdARAgQIECBAgEDuAgL9uRNrgAABAgQIECDQGALL126MZWs3xUMvLI3fP788G/Thu+4QR+wyLIb27xM79PdPz8a4EoyyuwXci909A9onQIAAAQIECFRfwP9tVd9ciwQI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4F5s38VZAtsjUG4MdMGCBdvTXN2XbfhAf/pGdvTo0bFkyZJsJX7r/xlNq/PnzZsXQ4YMiREjRnR4MaQg/0svvRT//d//HQ8++GCce+65cdZZZ5W0L3+HFUskQIAAAQIECBAgQIAAAQIECBAgQIAAAQIdCHS+Z00Hheslaffdd4/Vq1dnq/pTwL5wpAfwPvHEEzFlypQYM2ZM4XS7r2n/2W9961vx8MMPxwUXXBDvfve7BfnblXKSAAECBAgQIECAAAECBAgQIECAAAECBLpSQKA/Igvkp4fl3nnnnfHiiy9mQf/0MN0ZM2ZkP8ccc0zL1j1plX8K6s+dOzc2bNiQzUV6veOOO+K+++7Ltus5/vjjszqWLl0ahZ/ly5dH6y8RunIS1UWAAAECBAgQIECAAAECBAgQIECAAAECjSvQ8Fv3pKlP+++/973vjcsvvzyuuOKKePOb35xt5XPPPffEPvvsE9OnT295SFvas/+b3/xmpKc8X3nllTFp0qSYP39+/PKXv8weuJu27/ne9763zRWV2njPe94TgwcP3ibNCQIECBAgQIAAAQIECBAgQIAAAQIECBAgUKmAQP/rcimg/6UvfSl+/OMfx6233pptu3PiiSfGO97xjpbV/Clrr169YtSoUTFhwoRoamrKSqfV/+mBvelJ0U899VS7c5G2/7Giv10aJwkQIECAAAECBAgQIECAAAECBAgQIEBgOwQE+l/HSwH8PfbYI7785S93yNnc3Byf+MQn2uSZOnVq3HjjjW3O+UCAAAECBAgQIECAAAECBAgQIECAAAECBKohYI/+aihrgwABAgQIECBAgAABAgQIECBAgAABAgQI5CQg0J8TrGoJECBAgAABAgQIECBAgAABAgQIECBAgEA1BAT6q6GsDQIECBAgQIAAAQIECBAgQIAAAQIECBAgkJOAQH9OsKolQIAAAQIECBAgQIAAAQIECBAgQIAAAQLVEBDor4ayNggQIECAAAECBAgQIECAAAECBAgQIECAQE4CAv05waqWAAECBAgQIECAAAECBAgQIECAAAECBAhUQ0CgvxrK2iBAgAABAgQIECBAgAABAgQIECBAgAABAjkJCPTnBKtaAgQIECBAgAABAgQIECBAgAABAgQIECBQDQGB/mooa4MAAQIECBAgQIAAAQIECBAgQIAAAQIECOQkINCfE6xqCRAgQIAAAQIECBAgQIAAAQIECBAgQIBANQQE+quhrA0CBAgQIECAAAECBAgQIECAAAECBAgQIJCTgEB/TrCqJUCAAAECBAgQIECAAAECBAgQIECAAAEC1RAQ6K+GsjYIECBAgAABAgQIECBAgAABAgQIECBAgEBOAgL9OcGqlgABAgQIECBAgAABAgQIECBAgAABAgQIVENAoL8aytogQIAAAQIECBAgQIAAAQIECBAgQIAAAQI5CQj05wSrWgIECBAgQIAAAQIECBAgQIAAAQIECBAgUA0Bgf5qKGuDAAECBAgQIECAAAECBAgQIECAAAECBAjkJCDQnxOsagkQIECAAAECBAgQIECAAAECBAgQIECAQDUEBPqroawNAgQIECBAgAABAgQIECBAgAABAgQIECCQk4BAf06wqiVAgAABAgQIECBAgAABAgQIECBAgAABAtUQEOivhrI2CBAgQIAAAQIECBAgQIAAAQIECBAgQIBATgIC/TnBqpYAAQIECBAgQIAAAQIECBAgQIAAAQIECFRDQKC/GsraIECAAAECBAgQIECAAAECBAgQIECAAAECOQkI9OcEq1oCBAgQIECAAAECBAgQIECAAAECBAgQIFANAYH+aihrgwABAgQIECBAgAABAgQIECBAgAABAgQI5CQg0J8TrGoJECBAgAABAgQIECBAgAABAgQIECBAgEA1BAT6q6GsDQIECBAgQIAAAQIECBAgQIAAAQIECBAgkJOAQH9OsKolQIAAAQIECBAgQIAAAQIECBAgQIAAAQLVEBDor4ayNggQIECAAAECBAgQIECAAAECBAgQIECAQE4CAv05waqWAAECBAgQIECAAAECBAgQIECAAAECBAhUQ0CgvxrK2iBAgAABAgQIECBAgAABAgQIECBAgAABAjkJCPTnBKtaAgQIECBAgAABAgQIECBAgAABAgQIECBQDQGB/mooa4MAAQIECBAgQIAAAQIECBAgQIAAAQIECOQkINCfE6xqCRAgQIAAAQIECBAgQIAAAQIECBAgQIBANQQE+quhrA0CBAgQIECAAAECBAgQIECAAAECBAgQIJCTgEB/TrCqJUCAAAECBAgQIECAAAECBAgQIECAAAEC1RAQ6K+GsjYIECBAgAABAgQIECBAgAABAgQIECBAgEBOAgL9OcGqlgABAgQIECBAgAABAgQIECBAgAABAgQIVENAoL8aytogQIAAAQIECBAgQIAAAQIECBAgQIAAAQI5CQj05wSrWgIECBAgQIAAAQIECBAgQIAAAQIECBAgUA0Bgf5qKGuDAAECBAgQIECAAAECBAgQIECAAAECBAjkJCDQnxOsagkQIECAAAECBAgQIECAAAECBAgQIECAQDUEBPqroawNAgQIECBAgAABAgQIECBAgAABAgQIECCQk4BAf06wqiVAgAABAgQIECBAgAABAgQIECBAgAABAtUQEOivhrI2CBAgQIAAAQIECBAgQIAAAQIECBAgQIBATgJ9c6pXtQQIECBAgAABAj1cYMmajbF09YZsFMMGNsXwAf7p2MOnVPcJlCTg3i+JSSYCBAgQIECAQE0J+L+1mpoOnSFAgAABAgQIdL/Ac6+uiecWr4k/v7QyZs5bmXVo350Hx347DY5JIwbEpJEDur+TekCAQJcLuPe7nFSFBAgQIECAAIGqCQj0V41aQwQIECBAgACB2he465nF8cM/LIhrH13QtrOPvvbxvIPGxrmHjI3jdxvRNt0nAgR6tIB7v0dPn84TIECAAAECBEKg30VAgAABAgQIECCQCaTVvNO/+6cONdIXAOln1qVHWNnfoZREAj1HwL3fc+ZKTwkQIECAAAECxQQ8jLeYjPMECBAgQIAAgQYTuOxXs0secTl5S65URgIEukWgnPu5nLzdMhiNEiBAgAABAgQaVECgv0En3rAJECBAgAABAq0FFq/eED/cerue1hm2ep/ypjIOAgR6toB7v2fPn94TIECAAAECBAoCAv0FCa8ECBAgQIAAgQYWuH/W0rJHX0mZshtRgACBXAUquY8rKZPrIFROgAABAgQIECAQAv0uAgIECBAgQIAAgZj50sqyFSopU3YjChAgkKtAJfdxJWVyHYTKCRAgQIAAAQIEBPpdAwQIECBAgAABAhHDB/Ytm6GSMmU3ogABArkKVHIfV1Im10GonAABAgQIECBAQKDfNUCAAAECBAgQIBBxzJThZTNUUqbsRhQgQCBXgUru40rK5DoIlRMgQIAAAQIECAj0uwYIECBAgAABAgQi9h03OKbsOKBkipQ3lXEQINCzBdz7PXv+9J4AAQIECBAgUBCwR39BwisBAgQIECBAoMEFLjthYskC5eQtuVIZCRDoFoFy7udy8nbLYDRKgAABAgQIEGhQAYH+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IND/umhasb/rrrvGhRde2KFxeoDWe97zng7zpMS00v/yyy/vNJ8MBAgQIECAAAECBAgQIECAAAECBAgQIEBgewQ8jHd79JQlQIAAAQIECBAgQIAAAQIECBAgQIAAAQLdLCDQ380ToHkCBAgQIECAAAECBAgQIECAAAECBAgQILA9AgL926OnLAECBAgQIECAAAECBAgQIECAAAECBAgQ6GYBgf5ungDNEyBAgAABAgQIECBAgAABAgQIECBAgACB7REQ6N8ePWUJECBAgAABAgQIECBAgAABAgQIECBAgEA3Cwj0d/MEaJ4AAQIECBAgQIAAAQIECBAgQIAAAQIECGyPgED/9ugpS4AAAQIECBAgQIAAAQIECBAgQIAAAQIEullAoL+bJ0DzBAgQIECAAAECBAgQIPD/2bsPMDmqM9//L6AcRxoJSSiOIkIiKSCyBAKEybZlsDFre9cR27ve612vsdfXiOv9G3uv9++Ic8JmMbCATEZECUQwSQTlMIOEcpwZ5QDc51ejalX3dPdUnY7V/a3n6ZnuqnPqnPpUV1fVW6dOIYAAAggggAACCCCQiwCB/lz0yIsAAggggAACCCCAAAIIIIAAAggggAACCCCAQIkFCPSXeAVQPAIIIIAAAggggAACCCCAAAIIIIAAAggggAACuQgQ6M9Fj7wIIIAAAggggAACCCCAAAIIIIAAAggggAACCJRYgEB/iVcAxSOAAAIIIIAAAggggAACCCCAAAIIIIAAAgggkIsAgf5c9MiLAAIIIIAAAggggAACCCCAAAIIIIAAAggggECJBQj0l3gFUDwCCCCAAAIIIIAAAggggAACCCCAAAIIIIAAArkIEOjPRY+8CCCAAAIIIIAAAggggAACCCCAAAIIIIAAAgiUWIBAf4lXAMUjgAACCCCAAAIIIIAAAggggAACCCCAAAIIIJCLAIH+XPTIiwACCCCAAAIIIIAAAggggAACCCCAAAIIIIBAiQUI9Jd4BVA8AggggAACCCCAAAIIIIAAAggggAACCCCAAAK5CBDoz0WPvAgggAACCCCAAAIIIIAAAggggAACCCCAAAIIlFiAQH+JVwDFI4AAAggggAACCCCAAAIIIIAAAggggAACCCCQiwCB/lz0yIsAAggggAACCCCAAAIIIIAAAggggAACCCCAQIkFCPSXeAVQPAIIIIAAAggg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EEAAAQQQQCAhQKA/QcEbBBBAAAEEEEAAAQQQQAABBBBAAAEEEEAAAQTiJ0CgP37rjBojgAACCCCAAAIIIIAAAggggAACCCCAAAIIIJAQINCfoOANAggggAACCCCAAAIIIIAAAggggAACCCCAAALxE2gXvypXXo03bdpkd999t73yyivewk2aNMlmzpxp/fr1q7yFZYkQKFOBHTt22A9+8ANbs2ZNqxr26dPH/vVf/9UGDhzYahojEEAgd4GdO3faz3/+c5syZYpNmzat1QwPHDhgL774oj344IO2YcMGGzBggF122WV2+umnW4cOHVqlZwQCCLgJtLUtzpkzx2677ba0M7/00kvtox/9aNppjEQAgbYFtm3bZg8//LC99NJL1tjYaDU1NXbGGWfYJZdc4r0PzoH9YlCD9wjkTyDKdsg+MX/uzAmBVAHtB5944gmbN2+et08cNmyYfeADHzDFS1PP/9gnJusdc8MNN8xKHsWnYgqsW7fOvv/979vatWvtnHPOMX15X3jhBXv55ZftxBNPtB49ehSzOpSFQNUKaFv8wx/+YMcee6yNGDHC+vbtm/Q66aSTrFu3blXrw4IjUCiBd9991x544AH7y1/+Yqeeeqodf/zxSUVp+v3332+/+c1vbPjw4XbaaafZnj17bPbs2daxY0cbM2aMHX00NygmofEBAQeBMNuigpBvvvmmd4zav3//pP2kjmFHjRrlUDJZEEBAx6Hf+973vIZfOgfUhW/t25588kl7/fXXTceh3bt396DYL/J9QaAwAlG3Q/aJhVkPzBUBNcL88Y9/bM8884yNHz/eO/9rbm62e+65p9X5H/vE1t8XWvS3NinaGF11uuuuu0xf2K985StWV1fnlX3yySfbT3/6Uy+I8bnPfa7V1aqiVZCCEKgigS1bttjevXu91ojjxo1LWnKdaHHRLYmEDwjkRUD7QbWGuvXWW2337t1p57lkyRKvBbFacFx11VXWpUuXRKBfLYu1vaZeHEg7I0YigEBGgTDb4v79+23jxo3eBbl0x6edOnXKOH8mIIBAZoGDBw/aQw89ZPX19fbVr37VC2poe9q3b5/pvPCHP/yhPfroo/aJT3zCjjnmGGO/mNmSKQi4CkTdDtknukqTD4HsAu+//753kVt3t33+85+3s88+27RP1LmiWvKnnv+xT2ztSRO41iZFG7N+/Xqb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)

可见是对称的。即$h[n] = h[N-n-1]$。因此可以将乘法部分合并：

$$

h[n]x[k-n] + h[N-n-1]x[k-N+n+1] = h[n] (x[k-n]+x[k-N+n+1])

$$

得到下列结构：

![3bf7163eb0dae739f889cbc057c2df1b.png](en-resource://database/581:1)

这种方法可以half乘法器的数量，但是加法器的数量不变。同时，乘法器的reg数量变化很小。因此大致可以减少一半乘法器的硬件资源。其延时也减小约一半，为

$$

\frac{N}{2}+2

$$

  

### (5) 对称+转置型

可以结合上述(3)和(4)，得到下列结构：

  

![38bfc9d753dd9fa2e021975b833381ee.png](en-resource://database/582:1)

  

如此，延时减小为2拍，但是移位寄存器和加法器增加了$h[n]$的reg量。在抽头总数较大的情况下，不易于使用这种方法。综上，应该选择第四种方法。

  

### （6）方案总结

#### i. 最简单方法

如果采用**最简单**的FIR Filter设计方法，可以来计算一下这套方案（已经加入Polyphase）的开销（96阶，8行）：

1. Shift Register：$8 \times 11 \times 10_{bit} = 880$

2. h[n] coefficient：$8 \times 12 \times 12_{(bit)} = 1152$

3. Mutiplier：$8 \times 12 \times (10+12)_{(bit)} = 2112$

4. Adder：$8 \times 11 \times 23_{(bit)} = 2024$

  

其中，乘法器因为h[n]的系数是12bit，信号是10bit，因此为了保存所有精度，将寄存器位数设定为(12+10)bit。加法器设置于乘法器之后，为了预留进位，因此在原来乘法器输出位数基础上+1，设定为23bit。已知采用的工艺每个这样的部分所占面积为$3.75 \mu m^2$。最后计算总代价（因为有IQ两路，还需要$\times2$）：

* Summary：$（880+1152+2112+2024） \times 3.75 \mu m^2 = 46260 \mu m^2$

  

这是一个非常大的面积，占了整个系统面积将近21%（假如说芯片是$1mm \times 1mm$），这还只是滤波器所占的，后续的DSP还会占据更多的位置，采用这样的方案明显是不可行的。因此选择优化FIR Filter。

  

#### ii. 优化方案

在上一节的最后认为应该采用对成型滤波器的方案，已知FIR Filter的冲激响应是对称的，来计算下哪些h[n]的系数是相同的。（在此要声明，虽然理论上看来FIR Filter是96阶的，但实际不为0的h[n]只有95个，另外加上1个等于0的h[95]主要是为了实现设计8个sub-FIR Filter的目标。）

若，

$$

h[8m_1+r_1] = h[8m_2+r_2]

$$

则，

$$

\begin{aligned}

8(m_1+m_2)+(r_1+r_2) &= 94, \ (r_1+r_2 \leq 14)\\

\Rightarrow \begin{cases}

m_1+m_2 = 11\\

r_1+r_2 = 6

\end{cases} &or

\begin{cases}

m_1+m_2 = 10\\

r_1+r_2 = 14

\end{cases}

\end{aligned}

$$

  

可以将这个结果图形化：

![900b44170aaa02974933983f1ecf054b.png](en-resource://database/583:1)@w=600

可以看到除了$r=7$这一行外，其余的h[n]都关于点（5.5,3）中心对称（当然5.5这个值并不实际存在）。那么我们就可以将h[n]的系数个数直接减半，如果更加理想的话，就可以只用48个卷积核去得到滤波结果：

![039fca198e332ccf164b4d1e14cd21e3.png](en-resource://database/584:1)@w=800

  

这个图的意思是比如说当x[n]移至$m=11$的位置时，它所乘以的系数总体正好和x[n+11]相等，只是x[n]在sub-FIR[k]中相乘的系数和x[n+11]在sub-FIR[6-k]中相乘的系数相等，因此可以抽象地看成相乘“顺序相反”，所以x[n]和x[n+11]的箭头方向是相反的。

  

但是让x[n]和x[n+11]这两个信号同时进入一个Multiplier中进行运算显然是无法做到的，如果硬要做的话，假如说基带信号是400MHz，要么采用一个频率为200MHz的FIR Filter，要么采用一个频率为800MHz的FIR Filter

  

  

  

# 五、数字综合以及Cadence仿真验证

## 11/18

目前已经数字综合完成！

需要做的是以下部分：

- [x] 运行A文件后，运行B文件，生成一套完整的码字（包括I/Q两路的org_seq, ny_seq, y_k），并且**这套码字不要动**，存到文件reference下

- [x] 同时跑一遍modelsim仿真，生成完整的output i和output q，综合上述的几个文件，运行C文件。

- [x] 运行D文件生成.dat文件，把.dat文件super copy至端口中，随后将输出结果.csv文件导入到电脑中（i/q分开发）

- [ ] 运行E文件，对比时域结果，理应完全一致

- [ ] 运行F文件，对比频谱结果，理应完全一致

另外，在跑仿真的时候，可以做以下事情：

- [ ] 先前做的FIR有些观测口不可用，现在把bug修复好了，要重新丢进去综合一遍

按理来说，org_seq是500个点（一路），那么ny_seq是1000个点，对应输出应该是，每一个sub-FIR都是1000个点。

数值很接近，但是差得有点多，目前还没想好是什么问题， 但是感觉应该定位到各个sub-FIR上，所以首先，上述结果不要变。

## 11/19

随后综一个Sub-FIR，查询中间态的问题。先跑20个点100ns。

- [x] 综一个sub-FIR_0

- [x] 先check data_in没有问题

- [X] 检查data_out ,**有问题，最小bit不太对**

- [x] 检查乘法器，**有问题，问题定位到乘法器上**

综一个版本的乘法器出来，检查乘法器的问题

- [x] 综一个mult，**发现结果都正确**

另外后续debug要注意的有：

- [x] 考虑时钟相位的问题，此时正确了

  

  

**他妈的就是没考虑“沿采沿”的问题！**

  

  

  

## GTA Library/数字综合文件记录

### Libraries

* DTX_CJR_TOPFIR_NOV_19：用于仿真的模块，观测口是正确输出的，有问题，是LVS过不了，需要重新PR

- [ ] 对应tb，DTX_CJR/tb_top_FIR_test_NOV_19

* DTX_CJR_TOPFIR_NOV_19_2：新添了verilogA的采样DFF，输出结果就没有杂乱的结果

### 端口仿真

#### 17:10