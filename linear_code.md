# Introduction

[Linear code](https://en.wikipedia.org/wiki/Linear_code) can be considered as a function $C: F_q^k -> F_q^n$ that satisfies:

$$
C(ax_1 + bx_2) = aC(x_1) + bC(x_2)
$$

where $a,b \in F_q, x_1, x_2 \in F_q^k$.

We claim $C$ must be of the form: 

$$
C(x) = G x, x \in F_q^k
$$

where G is a `n x k` matrix.

Why ?

# Principle




$$

\begin{align*}
& \forall x = \begin{bmatrix} x_1 & x_2 & ... & x_k \end{bmatrix}^\top \in F_q^k\\
C(x) & = C(\begin{bmatrix} x_1 & x_2 & ... & x_k \end{bmatrix}^\top) \\
       & = C(x_1\begin{bmatrix} 1 & 0 & ... & 0\end{bmatrix}^\top + x_2\begin{bmatrix} 0 & 1 & ... & 0\end{bmatrix}^\top + ... + x_k\begin{bmatrix} 0 & 0 & ... & 1\end{bmatrix}^\top)\\
       & = x_1C(\begin{bmatrix} 1 & 0 & ... & 0\end{bmatrix}^\top) + x_2C(\begin{bmatrix} 0 & 1 & ... & 0\end{bmatrix}^\top) + ... + x_kC(\begin{bmatrix} 0 & 0 & ... & 1\end{bmatrix}^\top)\\
       & = x_1\begin{bmatrix} a_{11} & a_{12} & ... & a_{1n} \end{bmatrix}^\top + x_2\begin{bmatrix} a_{21} & a_{22} & ... & a_{2n} \end{bmatrix}^\top + ... + x_k\begin{bmatrix} a_{k1} & a_{k2} & ... & a_{kn} \end{bmatrix}^\top\\
       & = \begin{bmatrix} 
              a_{11} & a_{21} & ... & a_{k1} \\
              a_{12} & a_{22} & ... & a_{k2} \\
              ... \\
              a_{1n} & a_{2n} & ... & a_{kn} \\
              \end{bmatrix} \begin{bmatrix} x_1 & x_2 & ... & x_k \end{bmatrix}^\top\\
       & = \begin{bmatrix} 
              a_{11} & a_{21} & ... & a_{k1} \\
              a_{12} & a_{22} & ... & a_{k2} \\
              ... \\
              a_{1n} & a_{2n} & ... & a_{kn} \\
              \end{bmatrix} x

\end{align*}\\

$$