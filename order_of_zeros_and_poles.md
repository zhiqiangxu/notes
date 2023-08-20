# Introduction

For rational functions defined on elliptic curve, there's an important concept called **order**, which is motivated by multiplicities of zeros in analysis of functions in one variable. It's the basis for the more advanced concept of **divisor**, which is used in **pairing**.

Put simply, the order of a rational function `r` at a specific point `P` on elliptic curve `E`, is `d` iff:

$$r=u^{d} \cdot s$$

where `u` is a **uniformizer** and `s(P)` is finite and non zero.

The uniformizer `u` at `P` is a function $\in K(E)$ with $u(P)=0$ that also satisfies the following property:

$\forall r \in K(E)\backslash\lbrace0\rbrace$,  $\exists d \in Z, s \in K(E)$ finite and non zero at P, s.t. $r=u^d \cdot s$

We'll notate it as

$$order_P(r) := d$$

It turns out `d` doesn't depend on which uniformizer is chosen.

And $\forall P \in E$, there're 3 important facts about uniformizer which will be used frequently:

1. If `P` is finite and note of order two, then for `P=(a, b)`, the function $u(x,y):=x-a$ is a uniformizer at `P`.
2. If `P` is of order two, then $u(x,y):=y$ is a uniformizer at `P`.
3. If `P` is the infinite point, then $u(x,y):=\frac{x}{y}$ is a uniformizer at `P`.

Enough theory, let's dive into an example.

# Example

Let's suppose the equation for elliptic curve `E` is:

$$y^2 = x^3 -1$$

If we want to calculate the order of $r(x,y)=x-1$ at point $P=(1,0)$.

We notice `P` is of order `2` since its y coordinate is `0`.

So we choose the uniformizer $u(x,y):=y$.

Let's denote the other two roots of $x^3 -1=0$ as $\omega_2, \omega_3$.

Then $r(x,y)=x-1=\frac{(x-1)(x-\omega_2)(x-\omega_3)}{(x-\omega_2)(x-\omega_3)}=\frac{y^2}{(x-\omega_2)(x-\omega_3)}=u^2\cdot s$

Thus $order_P(x-1) = 2$

If we want to calculate the order of $r(x,y)=y$ at point $P=(1,0)$.

Since $r(x,y)$ itself is a uniformizer at `P`, and $r(x,y)=r(x,y)^1 \cdot 1 = u^d \cdot s$

Thus $order_P(y) = 1$

# References
1. https://www.wallenborn.net/download/Talk-Algebraic-Methods-in-Computation-Complexity-Elliptic-Curves-Divisors-and-Lines.pdf
