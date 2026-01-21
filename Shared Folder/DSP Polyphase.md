对于FIR插值滤波器，其Interpolation Factor = L，如果input freq. = $f_s$，则output freq. = $L\times f_s$，先从数学层面理解：
***
Parameters：

**Tap Coefficients**:$\ N$

**Original Samples**:$\ K$

**input**: $\ x[n] \overset{z}\leftrightarrow X[z]$

**interpolation factor**:$\ L$

**FIR impulse response**:$\ h[n]/H(z)$

**target output**:$\ y[z]/Y(z)$

**input after interpolating zero:$\ x_{(L)}[n]/X_{{L}}(z)$
***
Step 1. interpolation.** 在对原始输入信号进行插值后可以得到新的输入为，

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
**Step 2. Filter.** 进行FIR滤波，
$$

y[n] = h[n]*x_{(L)}[n]

$$
==时域卷积相当于频域相乘==，再将Step1中结果代入可得，
$$

Y(z) = H(z)X_{(k)}(z) = H(z)X(z^L).

$$
已知Tap Coefficient为N（i.e.只有在0~N-1范围内$h[n]$才不为0，因此省略掉这个范围外的$h[n]$）
$$
\begin{aligned}

H(z) = \sum_{n=-\infty}^{\infty} h[n]z^{-n} =\sum_{n = 0}^{N-1} h[n]z^{-n}\\

X(z^L) = \sum_{n=-\infty}^{\infty} h[n]z^{-n} =\sum_{k = 0}^{K-1} x[k]z^{-kL}\\

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

y[n] = \sum_{k=0}^{K-1} x[k] \cdot h[n-Lk]

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

仔细观察以上顺序（$h[n]$是关于$x_{(L)}[n]$的FIR冲激响应），$y[0]$~$y[7]$每个相位都相差一个插值后的间隔。$y[0]$、$y[8]$、$y[16]$可以看成插值前$x[0]$、$x[1]$、$x[2]$和FIR冲激响应卷积后的结果。另外同理，$y[8]$~$y[15]$每个点相位相差一个插值后的间隔。

  

那就可以得出结论，我们需要将FIR滤波器的冲激响应分成$L$组，以方便对应相移要求。另外每组前后两个冲激响应应该差L个样本点。（具体看图吧，实在太乱了）

![111de783e813e823935a2d96fe6103b4.png](en-resource://database/597:1)

r为定值的每一行对应一种间隔的相移（相移实际在Serializer种完成），为了防止歧义（我之前理解错误过），给出一组sub-FIR的真正示意图像（并非我们真正使用的，只是示意），

![3ea00bfe45fa749d90c57ef787ee7b8e.png](en-resource://database/598:1)@w=600

***