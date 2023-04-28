# Introduction
`fft` is the best algorithm to evaluate a lower-than-N-degree polynomial on `N` $N^{th}$ root of unity so far. I regard it as a great result of ingenious observation!

The core idea behind it is: divide and conquer. For a problem of size `N`, divide into multiple problems of subsize and combine them, avoiding as much superfluous computation as possible.

We'll restrict `N` to be a power of two herein, as it's the actually used case in ZKP.

# Principle

For a problem of size `N`, let's suppose there's a polynomial:

$$f(x) = c_0 + c_1x + c_2 x^2 + ... + c_{N-1}x^{N-1}$$


We want to compute $f(\omega_N^k)$ for $k \in [0, N)$, where $\omega_N$ is a primitive $N^{th}$ root of unity. So `N` evaluations in total.

There're mainly two ways to divide the problem:
1. Divide by even and odd coefficients, resulting in $(c_0, c_2, ..., c_{N-2})$ and $(c_1, c_3, ..., c_{N-1})$, we denote it by `DIT`(decimation in time).
2. Dive by first and second half, resulting in $(c_0, c_1, ..., c_{N/2-1})$ and $(c_{N/2}, c_{N/2+1}, ...,, c_{N-1})$, we denote it by `DIF`(decimation in frequency).

Notation:
1. $(c_0, c_1, ..., c_{N-1}) \coloneqq c_0 + c_1x + c_2 x^2 + ... + c_{N-1}x^{N-1}$, and so on.

For `DIT`, $\forall k \in [0, N/2)$:

$$\begin{equation}\begin{aligned} 
                  f(k) & = \sum_{i=0}^{N-1} c_i \omega_N^{ik} \\ 
                       & = \sum_{i=0}^{N/2-1} c_{2i} \omega_N^{2ik} + \sum_{i=0}^{N/2-1} c_{2i+1} \omega_N^{(2i+1)k} \\ 
                       & = \sum_{i=0}^{N/2-1} c_{2i} \omega_{N/2}^{ik} + \omega_N^k\sum_{i=0}^{N/2-1} c_{2i+1} \omega_{N/2}^{ik}\\            
              f(k+N/2) & = \sum_{i=0}^{N/2-1} c_{2i} \omega_{N/2}^{i(k+N/2)} + \omega_N^{k+N/2}\sum_{i=0}^{N/2-1} c_{2i+1} \omega_{N/2}^{i(k+N/2)}\\
                       & = \sum_{i=0}^{N/2-1} c_{2i} \omega_{N/2}^{ik} - \omega_N^k\sum_{i=0}^{N/2-1} c_{2i+1} \omega_{N/2}^{ik}
\end{aligned}\end{equation}$$

For `DIF`, $\forall k \in [0, N)$:

$$\begin{aligned} 
                  f(k) & = \sum_{i=0}^{N-1} c_i \omega_N^{ik} \\ 
                       & = \sum_{i=0}^{N/2-1} c_i \omega_N^{ik} + \sum_{i=N/2}^{N-1} c_{i} \omega_N^{ik} \\ 
                       & = \sum_{i=0}^{N/2-1} c_i \omega_N^{ik} + \sum_{i=0}^{N/2-1} c_{i+N/2} \omega_N^{(i+N/2)k} \\ 
                       & = \sum_{i=0}^{N/2-1} c_i \omega_N^{ik} + \omega_N^{kN/2}\sum_{i=0}^{N/2-1} c_{i+N/2} \omega_N^{ik} \\ 
                       & = \sum_{i=0}^{N/2-1} c_i \omega_N^{ik} + (-1)^k\sum_{i=0}^{N/2-1} c_{i+N/2} \omega_N^{ik} \\ 
                       & = \sum_{i=0}^{N/2-1} (c_i + (-1)^kc_{i+N/2})\omega_N^{ik}
\end{aligned}$$

So $\forall k \in [0, N/2)$:

$$\begin{equation}\begin{aligned} 
                  f(2k)     & = \sum_{i=0}^{N/2-1} (c_i + c_{i+N/2})\omega_N^{2ik}\\
                            & = \sum_{i=0}^{N/2-1} (c_i + c_{i+N/2})\omega_{N/2}^{ik}\\
                  f(2k+1)   & = \sum_{i=0}^{N/2-1} (c_i - c_{i+N/2})\omega_{N}^{i(2k+1)}\\
                            & = \sum_{i=0}^{N/2-1} (c_i - c_{i+N/2})\omega_{N}^i\omega_{N/2}^{ik}\\
\end{aligned}\end{equation}$$

Looking carefully we observe that after `1` iteration, the original problem of size `N` has been divided into two subproblems of size `N/2` for both `DIT` and `DIF`!

After $log_2^N-1$ iterations, we can imagine that the size `N` problem is divided into `N/2` problems of size `2`, which are trivial to solve.

But we're not done yet, details are the evil, let's dive into the implementation.

# Implementation

There're two big kinds of implementation:
1. out of place, where the evaluations are stored separately.
2. in place, where the evaluations are stored right in the place of coefficients.

The out-of-place case is trivial, just apply the reduction formula for `DIT` or `DIF` directly, e.g.:

```python
def fft(vals, modulus, domain):
    if len(vals) == 1:
        return vals
    L = fft(vals[::2], modulus, domain[::2])
    R = fft(vals[1::2], modulus, domain[::2])
    o = [0 for i in vals]
    for i, (x, y) in enumerate(zip(L, R)):
        y_times_root = y*domain[i]
        o[i] = (x+y_times_root) % modulus
        o[i+len(L)] = (x-y_times_root) % modulus
    return o
```
(quoted from [vitalik's blog](https://vitalik.ca/general/2019/05/12/fft.html), applying the `DIT` reduction formula)

This is great for demonstrating purpose, as it matches the reduction formula closely.

But the performance sucks, so for production usage one always prefers the in-place one.

Let's refer to the result after applying the reduction formula `n` times as layer `n`, so the last layer is layer $log_2^N-1$, where the problem size has shrinked to `2`, and the original size `N` problem is layer `0`.

Looking carefully at the reduction formula for `DIT` and `DIF` above, we observe that:
1. For `DIT`, we should apply the reduction formula from last layer to layer `0`.
2. For `DIF`, we should apply the reduction formula from layer `0` to the last layer.

But there's one more key observation not mentioned yet: 
1. For `DIT`, where does $c_i$ of the layer `0` go in the last layer? 
2. For `DIF`, where does $f(i)$ go in the last layer?

Looking really carefully this time, we observe that:
1. For `DIT`, the first iteration moves the **coefficient** with index $b_{log_2^N-1}...b_2b_10$(binary form) to index $0b_{log_2^N-1}....b_2b_1$, but moves the coefficient with index $b_{log_2^N-1}....b_2b_11$ to index $1b_{log_2^N-1}....b_2b_1$; The second iteration moves the coefficient with index $b_0b_{log_2^N-1}...b_2b_1$ to index $b_0b_1b_{log_2^N-1}...b_2$; And so on, the last iteration moves the in the end moves the coefficient with index $b_{log_2^N-1}...b_2b_1b_0$ of layer `0` to index $b_0b_1...b_{log_2^N-1}$, which is exactly the [bit reversal permutation](https://en.wikipedia.org/wiki/Bit-reversal_permutation) of the original index!
2. For `DIF`, a similar process happens, in the end, the **evaluation** for $b_{log_2^N-1}...b_2b_1b_0$ goes to index $b_0b_1b_2...b_{log_2^N-1}$!

So for `DIT` to apply reduction formula from last layer, the coefficients first need to apply the bit reversal permutation to get the expected result; For `DIF`,the coefficients are expected to be in order, but the final results need to apply the bit reversal permutation to get the expected result.

Hopefully you agree with me that it is a great result of ingenious observation!

With this in mind, one should be able to understand the implementation [here](https://github.com/AleoHQ/snarkVM/blob/ee36077d50836ac17d2406c2cc52b27706d29ec9/algorithms/src/fft/domain.rs#L161).

# References
1. http://dsp-book.narod.ru/DSPCSP/19.pdf
2. https://www.ii.uib.no/saga/SC96EPR/fft.html
3. https://www.cmlab.csie.ntu.edu.tw/cml/dsp/training/coding/transform/fft.html