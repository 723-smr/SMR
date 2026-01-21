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
**Step 1. interpolation.** 在对原始输入信号进行插值后可以得到新的输入为，

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
