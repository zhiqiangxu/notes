# Introduction

`zkp` has a key component called [`FRI`](https://eprint.iacr.org/2022/1216.pdf)(Fast, Reed-Solomon Interactive Oracle Proof of Proximity), which has something to do with `FFT`/`Reed-Solomon`/`IOPP`. We'll uncover `Reed-Solomon` in this post.

`Reed-Solomon` is one of the most frequently used error-correcting codes. Its notation is `RS(n, k, n-k+1)`, which means the message word has size `k`, the code word has size `n`, and the minimum hamming distance is `n-k+1`, where `k < n`. 

It has two variants:
1. `The original view`, the code word as a sequence of values.
    1. evaluations at `n` fixed points of a polynomial with degree less than `k`.
2. `The BCH view`, the code word as a sequence of coefficients.
    1. $c(x) = m(x)g(x)$, where $$\deg m(x) < k, \deg c(x) < n, g(x)=\prod_{i=r}^{i=r+n-k-1} (x-\omega^i), r \in [0, k]$$
    2. $m(x)$ is the message polynomial, $c(x)$ is the code polynomial, $g(x)$ is the generator polynomial, $\omega$ is a $n^{th}$ primitive root of unity.

Why does it work? Because it has the minimum hamming distance `n-k+1`, which means the minimum distance between any two codes is `n-k+1`, thus it can always correct up to `(n-k)/2` errors or `n-k` erasures(errors with known locations).

So the key point is to understand why the minimum hamming distance is `n-k+1` for both variants.

# Principle

It's not hard to see both variants are [linear code](https://en.wikipedia.org/wiki/Linear_code).

For `The original view`, it's trivial since for two polynomial of degree less than `k`, they can have at most `k-1` common evaluations, which means they have at least `n-k+1` different evaluations, thus the minimum hamming distance is at least `n-k+1`. As the [Singleton bound](https://en.wikipedia.org/wiki/Singleton_bound) of linear code is at most `n-k+1`, the minimum hamming distance can only be `n-k+1`.

For `The BCH view`, since it's also linear code, we only need to prove the minimum weight of nonzero code is `n-k+1`. 

If there's a code polynomial $c(x)$ with weight less than or equal to `n-k`, let's assume 

$$c(x) = a_{i_0}x^{i_0} + a_{i_1}x^{i_1} + ... + a_{i_{n-k-1}}x^{i_{n-k-1}}$$

We want to prove $c(x) = 0$.

Since $g(x)$ divides $c(x)$, $c(x)$ has to evaluate to zero at points $\omega^{r}, \omega^{r+1}, ..., \omega^{r+n-k-1}$:

$$a_{i_0}(\omega^{r})^{i_0} + a_{i_1}(\omega^{r})^{i_1} + ... + a_{i_{n-k-1}}(\omega^{r})^{i_{n-k-1}} = 0$$
$$a_{i_0}(\omega^{r+1})^{i_0} + a_{i_1}(\omega^{r+1})^{i_1} + ... + a_{i_{n-k-1}}(\omega^{r+1})^{i_{n-k-1}} = 0$$
...
$$a_{i_0}(\omega^{r+n-k-1})^{i_0} + a_{i_1}(\omega^{r+n-k-1})^{i_1} + ... + a_{i_{n-k-1}}(\omega^{r+n-k-1})^{i_{n-k-1}} = 0$$

The above is a system of `n-k` equations with `n-k` variables $\{a_{i_j}\}_{j \in [0, n-k-1]}$.

The determinant of the matrix 

$$ \begin{bmatrix} 
   (\omega^{i_0})^{r} & (\omega^{i_1})^{r} & ... & (\omega^{i_{n-k-1}})^{r} \\
   (\omega^{i_0})^{r+1} & (\omega^{i_1})^{r+1} & ... & (\omega^{i_{n-k-1}})^{r+1} \\
   ... & ... & ... & ... \\
   (\omega^{i_0})^{r+n-k-1} & (\omega^{i_1})^{r+n-k-1} & ... & (\omega^{i_{n-k-1}})^{r+n-k-1} \\
   \end{bmatrix} $$

can be shown to be nonzero as a property of the [Vandermonde Determinant](https://en.wikipedia.org/wiki/Vandermonde_matrix#Determinant), therefore the solution must be $$a_{i_0} = a_{i_1} = ... = a_{i_{n-k-1}} = 0 $$

$\mathbb{QED}$

