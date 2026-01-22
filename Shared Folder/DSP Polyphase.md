# 一、为什么要设计Polyphase Filter

## （1）离散序列的傅里叶变换

一个离散序列可以看成对一个连续波形进行采样后的结果，也就是在时域中用一个周期冲激序列乘以一个连续波形：
$$
\begin{aligned}
\delta_T(t)&=\sum_{n=-\infty}^{\infty}\delta(t-nT_s),\\
f_s(t)&=f(t)\delta_T(t).
\end{aligned}
$$
时域相乘，频域卷积：
$$
\begin {aligned}
\mathscr{F}[f_s(t)] &= F_s(\omega) = \frac{1}{2\pi} F(\omega) * \frac{2\pi}{T_s} \sum_{n=-\infty}^{\infty} \delta(\omega - n\omega_s) = \frac{\omega_s}{2\pi} \sum_{n=-\infty}^{\infty} F(\omega-n\omega_s),\\
i.e.\, \, \, \, \, F_s(\omega) &= \frac{\omega_s}{2\pi} \sum_{n=-\infty}^{\infty} F(\omega-n\omega_s).
\end {aligned}
$$
也就是原本一个连续波形的频谱在频域中被周期延拓了，如下图所示：
（放赛灵思手册的第一张图）

对于这个离散序列的频谱来说，这些镜像的频谱不造成影响，因为DFFT只关注第一奈奎斯特区的频谱。但是对离散序列进行ZOH后，镜像谱的影响就被体现出来了。
## （2）ZOH的后果

对一个离散序列进行ZOH，相当于用一个脉宽和采样周期一致的方波去卷积上这个周期序列：
$$
\begin{aligned}
	y_{ZOH} = \sum_{n}x[n][u(t-nT_s) - u(t-(n+1)T_s)]
\end{aligned}
$$
时域卷积等于频域相乘，一个方波的傅里叶变换为：
$$
	F(f) = E\tau Sinc(f\tau) 
$$
当原本的频谱中乘以一个sinc杂散后，高频的spurs就全部被显现出来了，根据通信要求，这些高频信号是不被允许存在的。因此如果要在shu




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
其中，$y[0]、y[8]、y[16]$这样几个点之间的间隔才符合插值前的基带频率。

(1)式其实是插值加滤波后的最终结果，因此已经加入了串行器。但我想知道为什么要将滤波器分为$L$组，因此我需要串行器因素仅仅看FIR Filter的仔埔平面给。假如说$N=160$，来举个例子看看，
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

## （1）FIR Filter的行为级

那就可以得出结论，我们需要将FIR滤波器的冲激响应分成$L$组，以方便对应相移要求。另外每组前后两个冲激响应应该相差一个采样周期（在例子中为$\frac{1}{400 MHz}$），比如说$y[0]$和$y[8]$在时域上相差一个采样周期。

给出一组FIR的示意图像如下，图中相同颜色的为同一组sub-FIR。同时，需要特别关注的是FIR的图像是对称的，图中红色框出来的FIR的抽头系数是相同的（即$h[0]$=$h[64]$)，所以为了减少资源，在下面的操作中会进行简化。
![](assets/DSP%20Polyphase/file-20260122113843089.png)
![](assets/DSP%20Polyphase/file-20260122111952015.jpg)

## （2） **FIR Filter和Serializer的逻辑**

先给出基带信号频率和Serializer时钟频率的关系表达式：
$$
	f_{up} = L \times f_{bb} 
$$
$y[nL]$~$y[(n+1)L - 1]$每个序列都相差$\frac{1}{L\times f_{bb}}$，$y[nL]$ 和 $y[(n+1)L]$ 相差的是$\frac{1}{f_{bb}}$。
***
>**e.g.**

已知基带信号为400 MHz，$L=8$，且时钟上升沿有效。只分析一个基带信号周期内的信号处理过程。

- $CLK_{bb}$的第一个周期：
1. 当$CLK_{bb}$的第一个上升沿到来时，$x[0]$进入Polyphase Filter，此时8个 sub-FIR 分别输出$y[0]$~$y[7]$ ; 
2. $y[0]$~$y[7]$并行进入Serializer；
3. 当$CLK_{up}$的第一个上升沿到来时，Serializer输出 $y[0]$；
4. $CLK_{up}$的第二个上升沿到来时，Serializer输出 $y[1]$；
5. ...

>第一个周期的8个 sub-FIR 输出：
$$

\begin{aligned}
y[0] &= x[0]h[0],\\
y[1] &= x[0]h[1],\\
...\\
y[7] &= x[0]h[7].\\

\end{aligned}
$$

- $CLK_{bb}$的第二个周期：
1. 当$CLK_{bb}$新的上升沿到来时，以 sub-FIR1 为例，$x[1]$也进入Filter，和$h[0]$相乘，此时$x[0]$往后推一位，和$h[8]$相乘，得到输出 $y[8]$；
2. $y[8]$~$y[15]$并行进入Serializer；
3. 当$CLK_{up}$的第一个上升沿到来时，Serializer输出 $y[8]$；
4. $CLK_{up}$的第二个上升沿到来时，Serializer输出 $y[9]$；
5. ...

>第二个周期的8个sub-FIR输出：
$$
\begin{aligned}
y[8] &= x[1]h[0]+x[0]h[8],\\
y[9] &= x[1]h[1]+x[0]h[9],\\
...\\
y[15] &= x[1]h[7]+x[0]h[15].
\end{aligned}
$$

$y[0]$~$y[7]$ 以3.2GHz的频率输出，等到第二个基带周期到来，$y[8]$~$y[15]$ 同样也以3.2GHz的频率输出，以此类推。这样就获得了插零插值+低通滤波的双重效果。
***
**注意：** 
* 相邻$y[k]$间隔一个基带周期，这种间隔实际在Serializer中完成（对应上图High-Speed Serializer中的$Z^{-1}$)，并不是在FIR中进行，在sub-FIR内$x[k]$与$h[k]$只是数值上进行的处理，并没有进行时延！
* 这里的滤波效果体现在插零的地方有了幅度：
*![](assets/DSP%20Polyphase/file-20260122134737404.png)
***

## （3） **FIR Optimization**
对于单径FIR，利用（1）中所说FIR滤波器图像的对称性，从而节省了一半的Mul.s：
![](assets/DSP%20Polyphase/file-20260122135912199.png)