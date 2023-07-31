# Introduction

In zkvm's constraint for logic operation, there're some heavily used tricks based on `even bit int`, which is unsigned integer with all odd bits zero(the LSB bit is $0^{th}$ bit and thus an even bit).

Let's denote `even bit int` as $E$, then this holds:

$$\forall a,b \in E, a+b = 2(a \land b) + (a \oplus b)$$

Let's prove it quickly.

First it's easy to see it holds for $\forall a,b \in \lbrace0, 1\rbrace$

So

$$
\begin{align*}
a + b &= (a_{2N}, ...,0, a_2, 0, a_0) + (b_{2N}, ...,0, b_2, 0, b_0) \\
      &= (a_{2N}\land b_{2N}, a_{2N}\oplus b_{2N},..,a_2\land b_2, a_2\oplus b_2, a_0\land b_0, a_0\oplus b_0) \\
      &= 2(a \land b) + (a \oplus b)
\end{align*}
$$

So it also holds for $\forall a,b \in E$.

Now it's time to extend the domain to arbitrary unsigned integer, let's first introduce a notation:

For $\forall a \in \mathbb{Z}^{+}, a = (a_{2N},..., a_1, a_0)$, define:

$$a_e := (a_{2N}..., 0, a_2, 0, a_0)$$

$$a_o := (a_{2N-1}..., 0, a_3, 0, a_1)$$

Obviously this holds:

$$a = 2a_o + a_e$$

what's more, for $\forall a,b \in \mathbb{Z}^{+}$, this also holds:

$$a \land b = 2(a_o+b_o)_o + (a_e + b_e)_o$$

$$a \oplus b = 2(a_o+b_o)_e + (a_e + b_e)_e$$

This may not be obvious at the first glance, so let's dive into the details.

# Principle

$$
\begin{align*}
a \land b &= 2(a \land b)_o + (a \land b)_e \\
          &= 2(a_o \land b_o) + (a_e \land b_e)\\
          &= 2(a_o + b_o)_o + (a_e + b_e)_o
\end{align*}
$$

$$
\begin{align*}
a \oplus b &= 2(a \oplus b)_o + (a \oplus b)_e \\
          &= 2(a_o \oplus b_o) + (a_e \oplus b_e)\\
          &= 2(a_o + b_o)_e + (a_e + b_e)_e
\end{align*}
$$

$\mathbb{qed}$