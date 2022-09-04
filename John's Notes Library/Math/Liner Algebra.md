# 线性代数
## 介绍
&nbsp;&nbsp;&nbsp;&nbsp;一个数学问题当可以简化成一个线性代数问题那就可以求解。线性代数问题可以最终简化成线性方程组的求解，涉及矩阵的运算。线性代数对于计算来说是至关重要的一个工具（或者说是各种工具的集合）。
## 向量空间和线性变换
&nbsp;&nbsp;&nbsp;&nbsp;定义1：一个集合$V$是实数集$\mathbb{R}$上的一个向量空间，如果存在映射：
1. $\mathbb{R}\times V\rightarrow V$，表示为$a\cdot\nu$或者$a\nu$对于所有的实数$a$和$V$中的元素$\nu$；
2. $V\times V\rightarrow V$，表示为$\nu+\omega$对于向量空间$V$中的所有元素$\nu$和$\omega$，有下列性质：
	1. 存在集合$V$中的元素0使得对于所有的$\nu\in V$有$0+\nu=\nu$；
	2. 对于所有的$\nu\in V$存在元素$(-\nu)\in V$且$\nu+(-\nu)=0$；
	3. 对于所有的$\nu,\omega\in V$，有$\nu+\omega=\omega+\nu$；
	4. 对于所有的$a\in\mathbb{R}$和$\nu,\omega\in V$，有$a(\nu+\omega)=a\nu+a\omega$；
	5. 对于所有的$a,b\in\mathbb{R}$和$\nu\in V$，有$a(b\nu)=(ab)\nu$；
	6. 对于所有的$a,b\in\mathbb{R}$和$\nu\in V$，有$(a+b)\nu=a\nu+b\nu$；
	7. 对于所有的$\nu\in V$，$1\nu=\nu$。

&nbsp;&nbsp;&nbsp;&nbsp;向量空间的元素称为向量，$\mathbb{R}$（或其他使用的域）中的元素被称为数量。向量空间之间的自然映射就是线性变换。

&nbsp;&nbsp;&nbsp;&nbsp;定义2：一个线性变换$T:V\rightarrow W$是从向量空间$V$到向量空间$W$的一个函数，使得对于任意实数$a_1，a_2$和$V$中的任意向量$\nu_1，\nu_2$有
$$T(a_1\nu_1+a_2\nu_2)=a_1T(\nu_1)+a_2T(\nu_2)$$

&nbsp;&nbsp;&nbsp;&nbsp;定义3：向量空间$V$的子集$U$是$V$的一个子空间，如果$U$本身是一个向量空间。

&nbsp;&nbsp;&nbsp;&nbsp;定义4：如果$T:V\rightarrow W$是一个线性变换，那么$T$的核是
$$ker(T) = \{\nu\in V\mid T(\nu)=0\}$$
$T$的像是
$$Im(T) = \{\omega\in W\mid存在\nu\in V使得T(\nu)=\omega\}$$
核是$V$的一个子空间，因为如果$\nu_1$和$\nu_2$是核中的两个向量，$a，b$是任意两个实数，那么
$$\begin{split}T(a\nu_1+b\nu_2) &= aT(v_1)+bT(\nu_2)\\
&= a\cdot0+b\cdot0\\
&= 0\end{split}$$
同理我们可以证明$T$的像是$W$的一个子空间。（$T$的全体像组成的集合称为$T$的值域，用$TV$表示. 所有被$T$变成零向量的向量组成的集合称为$T$的核，核和值域也记作$Ker(T)$和$Im(T)$）