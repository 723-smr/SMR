# 一、为什么要设计Polyphase Filter

## （1）




# 二、Polyphase数学推导

对于FIR插值滤波器，其Interpolation Factor = L，如果input freq. = $f_s$，
则output freq. = $L\times f_s$，先从数学层面理解：
***
### Parameters

- **Tap Coefficients**:$\ N$

- **Number of Original Samples**:$\ K$

- **Input**: $\ x[n] \overset{z}\leftrightarrow X[z]$

- **Input after IoZ**:$\ x_{(L)}[n]\overset{z}\leftrightarrow X_{{L}}(z)$

- **Interpolation Factor**:$\ L$

- **FIR Filter's Impulse Response**:$\ h[n]\overset{z}\leftrightarrow H(z)$

- **Output**:$\ y[z]\overset{z} \leftrightarrow Y(z)$
***
## （1）传统的先IoZ再低通滤波的方式
### Step 1. Interpolation.

在对原始输入信号进行插值后可以得到新的输入为，

$$

x_{(L)}[n] =

\begin{cases}

x[\frac{n}{L}], &n=0, L, 2L, ..., (K-1)L\\

0, &otherwise.

\end{cases}

$$

根据z变换时间扩展性质可以得到，
$$

X_{(L)}(z) = X(z^L).

$$
### Step 2. Low-pass Filter. 

进行FIR低通滤波，
$$

y[n] = h[n]*x_{(L)}[n]

$$
==时域卷积相当于频域相乘==，再将Step1中结果代入可得，
$$

Y(z) = H(z)X_{(k)}(z) = H(z)X(z^L).

$$

>**z变换时间扩展性质**
>
>若，
>$$

x_{(k)}[n] =

\begin{cases}

x[\frac{n}{k}], &n = 0,k,2k,... \\

0, &otherwise.

\end{cases}

$$
>则，
>$$

x[n] \overset{z}{\leftrightarrow} X(z),\\

x_{(k)}[n] \overset{z}{\leftrightarrow} X(z^k)

$$
## （2）Polyphase

已知Tap Coefficient为N（i.e.只有在0~N-1范围内$h[n]$才不为0，因此省略掉这个范围外的$h[n]$）
$$
\begin{aligned}

H(z) &= \sum_{n=-\infty}^{\infty} h[n]z^{-n} =\sum_{n = 0}^{N-1} h[n]z^{-n}\\

X(z^L) &= \sum_{n=-\infty}^{\infty} h[n]z^{-nL} = \sum_{n=0}^{K-1} h[n]z^{-nL}\\

\end{aligned}
$$

可以得到（为了便于辨认，将$x[n]$替换为$x[k]$），
$$\begin{aligned}

Y(z) &= \sum_{k = 0}^{K-1}x[k]z^{-kL} \cdot \sum_{n=0}^{N-1} h[n]z^{-n}\\

  

&=\{\sum_{k=0}^{K-1}x[k]\} \cdot z^{-kL}\{ \sum_{n=0}^{N-1} h[n]z^{-n}\}\\

  

&= \sum_{k=0}^{K-1}x[k] \cdot \{z^{-kL}\ H(z)\}

\end{aligned}$$
转换到时域可以得到，
$$

y[n] = \sum_{k=0}^{K-1} x[k] \cdot h[n-Lk] \tag{1}

$$
其中，$y[0]、y[8]、y[16]$这样几个点之间的间隔才符合插值前的滤波输出间隔。

(1)式其实是插值加滤波后的最终结果，因此已经加入了相移。但我想知道为什么要将滤波器分为$L$组，因此我需要排除相移因素仅仅看值的叠加。假如说抽头系数为160，来举个例子看看，
$$

\begin{aligned}

y[0] &= x[0]h[0],\\

y[1] &= x[0]h[1],\\

...\\

y[7] &= x[0]h[7],\\

y[8] &= x[0]h[8]+x[1]h[0],\\

...\\

y[16] &= x[0]h[16]+x[1]h[8]+x[2]h[0].

\end{aligned}

$$
仔细观察以上顺序（$h[n]$是关于$x_{(L)}[n]$的FIR冲激响应），$y[0]$、$y[8]$、$y[16]$...只和$h[0]$、$h[8]$、$h[16]$...有关，所以将$h[0]$、$h[8]$、$h[16]$...作为sub-FIR 0,依此类推。

# 三、Polyphase的工程解释

那就可以得出结论，我们需要将FIR滤波器的冲激响应分成$L$组，以方便对应相移要求。另外每组前后两个冲激响应应该差L个样本点。

给出一组FIR的示意图像如下，图中相同颜色的为同一组sub-FIR。同时，需要特别关注的是FIR的图像是对称的，图中红色框出来的FIR的抽头系数是相同的（即$h[0]$=$h[64]$)，
![](assets/DSP%20Polyphase/file-20260122113843089.png)
如下图所示，第i行作为sub-FIR i。
![](assets/DSP%20Polyphase/file-20260122113206934.png)
![](assets/DSP%20Polyphase/file-20260122111952015.jpg)

**上采样前后clock：**$y[0]$~$y[7]$每个相位都相差插零后一个的间隔，同理$y[8]$~$y[15]$每个点相位也相差插零后一个的间隔，依此类推。$y[0]$和$y[8]$之间相差的是插零前的一个间隔，即依次输入信号$x[k]$不同bit的间隔。
相邻$y[k]$对应一种间隔的相移，但相移实际在Serializer种完成（对应上图High-Speed Serializer中的$Z^{-1}$)。为了防止歧义，在sub-FIR内$x[k]$与$h[k]$只是数值上进行的处理，并没有进行时延！比如，当$x[0]$打入的时候
![](assets/DSP%20Polyphase/file-20260121213926710.png)
***