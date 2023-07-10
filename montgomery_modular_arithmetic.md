# Introduction

Modular arithmetic is often used in zkp, so today we're going to introduce the "Montgomery modular arithmetic" algorithm which is designed for fast modular arithmetic.

It uses a bijection `f` called "Montgomery form":

$$\forall a \in [0, N-1], f(a) \equiv aR \pmod{N} $$

where $R,N \in \mathbb{Z}^{+} \land gcd(R, N) \equiv 1 \land R > N$

(The constant $R$ is chosen so that division by $R$ is easy. In practice, $R$ is always a power of two, since division by powers of two can be implemented by bit shifting.)

So instead of doing modular arithmetic directly, this algorithm does it in Montgomery form.

Addition and subtraction in Montgomery form are the same as ordinary modular addition and subtraction because of the distributive law:

$$  aR \pmod{N} + bR \pmod{N} \equiv (a+b)R \pmod{N} $$

$$  aR \pmod{N} - bR \pmod{N} \equiv (a-b)R \pmod{N} $$

Multiplication in Montgomery form, however, is more complicated. The usual product of $aR\pmod{N}$ and $bR\pmod{N}$ does not represent the product of $a$ and $b$ because it has an extra factor of $R$:

$$(aR \pmod{N})(bR \pmod{N}) \pmod{N} \equiv (abR)R \pmod{N} $$

Computing products in Montgomery form requires removing the extra factor of $R$.

Removing the extra factor of $R$ can be done by multiplying $R^{-1}\pmod{N}$, which must exist because $gcd(R, N) = 1$.

A straightforward algorithm to multiply numbers in Montgomery form is therefore to multiply $aR \pmod N$, $bR \pmod N$, and $R^{-1}\pmod{N}$ as integers and reduce modulo N.

However, it is slower than multiplication in the standard representation because of the need to multiply by $R^{-1}\pmod{N}$ and divide by $N$.

Now it's time to introduce the Montgomery reduction, also known as REDC, which computes the same result as the naive method, but more quickly.

# The REDC algorithm

```
function REDC is
    input: Integers R and N with gcd(R, N) = 1,
           Integer N′ in [0, R − 1] such that NN′ ≡ −1 mod R,
           Integer T in the range [0, RN − 1].
    output: Integer S in the range [0, N − 1] such that S ≡ TR−1 mod N

    m ← ((T mod R)N′) mod R
    t ← (T + mN) / R
    if t ≥ N then
        return t − N
    else
        return t
    end if
end function
```

The REDC algorithm returns $TR^{-1}\pmod{N}$ for input $T$, where $T \in [0, RN-1]$.

To see that this algorithm is correct, first observe that $T+mN$ is a multiple of $R$:

$$T+mN \equiv T + (((T \pmod{R})N^{'}) \pmod{R})N \equiv T + TN^{'}N \equiv T - T \equiv 0 \pmod{R}$$

Therefore, `t` is an integer. Second, observe that `t` is congruent to $TR^{-1}\pmod{N}$:

$$t = (T+mN)/R \equiv (T+mN)R^{-1} \equiv TR^{-1} + mR^{-1}N \equiv TR^{-1}\pmod{N}$$

Therefore, the output has the correct residue class. Third, `m` is in `[0, R-1]` and therefore `T+mN` is between `0` and `RN-1+(R-1)N` < `2RN`. Hence t is in `[0,2N-1]`.Therefore, reducing `t` into the desired range requires at most a single subtraction.

Finally, $R > N$ guarantees $aR\pmod{N} bR\pmod{N}$ < $N^2 < RN$, and thus valid to apply REDC.

$\mathbb{qed}$
