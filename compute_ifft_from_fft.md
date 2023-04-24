# Introduction

One step of plonk needs to compute the quotient polynomial, which is a product of several polynomials divided by the vanishing polynomial.

If one computes this quotient polynomial directly, the complexity will be very high. But there's a trick: we can compute it in evaluation form instead of coefficient form, thus greatly reducing the complexity.

So the precondition is that we have computed the evaluation form of polynomials.

`fft` is an optimized algorithm to convert a polynomial from coefficient form to evaluation form. Its complexity is `O(nlog n)`. It converts a `n-1` degree polynomial `f(x)` into `n` evaluations on $n^{th}$ root of unity:

$$v_i = f(w^i) \text{ for }i\in[1, n], w\text{ is the primitive }n^{th}\text{ root of unity.}$$ 

Suppose $$f(x) = \sum_{k=0}^{k=n-1} c_k x^k$$

Then $$v_i = \sum_{k=0}^{k=n-1} c_k {w^i}^k$$

[$v_1, v_2, ..., v_{n}$] is the evaluation form of `f(x)` over `n` $n^{th}$ root of unity.

`ifft` is the reverse process of `fft`.

The implementation of `fft` is fairly complex, with no surprise. But the implementation of `ifft` is pretty simple:

```rust
/*
    a:          the evaluation form of target polynomial
    omega_inv:  the inverse of omega, the primitive nth root of unity
    divisor:    the inverse of n
 */
fn ifft(a: &mut [F], omega_inv: F, log_n: u32, divisor: F) {
    best_fft(a, omega_inv, log_n);
    parallelize(a, |a, _| {
        for a in a {
            // Finish iFFT
            *a *= &divisor;
        }
    });
}
```
(quoted from [halo2](https://github.com/zcash/halo2/blob/5678a506cbec60d593a21ff618d09bed48abf7f2/halo2_proofs/src/poly/domain.rs#L372))

It reuses the implementation of `fft` to compute `ifft`, which is neat! 

But why does it work? 

Let's dive into the details.

# Principle


Let's define an operation between two polynomials as follows:

$$\forall A(x), B(x) \in F[x], <A(x), B(x)> := (\sum_{i=1}^{i=n} {A(w^i)/B(w^i)})/n$$

where `F` is a field.

(Let's not worry about $B(w^i)=0$ for now, which is guaranteed below anyway.)

So $$\forall i,j \in Z, <x^i, x^j> \;=\; (\sum_{k=1}^{k=n} {w^{ik}/w^{jk}})/n \;=\; (\sum_{k=1}^{k=n} {w^{i-j}}^k)/n \;=\; \begin{cases} 1\text{ if } n\mid{i-j} \\ 
                                                                                                          0\text{ if } n\nmid{j-j}\end{cases}$$
So $$ <x^n, x^n> \;=\; 1$$

It's not hard to see that $$<\alpha A(x), B(x)>\; =\; \alpha <A(x), B(x)>$$

And let's suppose $$f(x) = \sum_{i=0}^{i=n-1} c_i x^i$$

So $\forall k\in [0,n)$, $$\begin{aligned}
    c_k & = <c_k x^k, x^k>\\ 
        & = <f(x), x^k>\\
        & =(\sum_{i=1}^{i=n} f(w^i)/{w^i}^k)/n\\
        & = (\sum_{i=1}^{i=n} v_i/{w^i}^k)/n
                        \end{aligned}$$

The above is exactly the same as computing the `fft` of the polynomial with coefficients [$v_1, v_2, ..., v_n$], but replace $w$ with $w^{-1}$ and then divide each evaluation by `n`.

The eagle eyes may ask, how did you come up with the idea of defining such a strange operation in the first place?

This idea has its root in representation theory, but it won't be covered here:)