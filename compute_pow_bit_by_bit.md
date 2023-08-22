# Introduction

It's Qixi Festival, today we'll quickly go over a code snippet which computes $a^b$ bit by bit.

Here's the code:

```rust
/// Exponentiates `self` by `exp`, where `exp` is a little-endian order integer
/// exponent.
///
/// # Guarantees
///
/// **This operation is variable time with respect to `self`, for all exponent.** If
/// the exponent is fixed, this operation is effectively constant time. However, for
/// stronger constant-time guarantees, [`Field::pow`] should be used.
fn pow_vartime<S: AsRef<[u64]>>(&self, exp: S) -> Self {
    let mut res = Self::ONE;
    for e in exp.as_ref().iter().rev() {
        for i in (0..64).rev() {
            res = res.square();

            if ((*e >> i) & 1) == 1 {
                res.mul_assign(self);
            }
        }
    }

    res
}
```

Why does it work?

# Principle

$$
\begin{aligned}
a^{\overline{b_1b_2...b_n}} &= a^{(\overline{b_1b_2...b_{n-1}})*2 + b_n}\\
                            &= (a^{\overline{b_1b_2...b_{n-1}}})^2 \cdot  a^{b_n}\\
                            &= ((a^{\overline{b_1b_2...b_{n-2}}})^2 \cdot a^{b_{n-1}})^2 \cdot  a^{b_n}\\
                            &= ...\\
                            &= ((((a^{b_1})^2 \cdot a^{b_2})^2 \cdot a^{b_3})^2 \cdot ...)^2 \cdot a^{b_n}
\end{aligned}
$$

By working from the innermost parentheses to the outermost parentheses, we can conclude the above code.

Really quick, as I promised:)

$\mathbb{qed}$