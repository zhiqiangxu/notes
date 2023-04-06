# Introduction

When doing arithmetization, we're actually doing 2 things:
1. correctly compute each witness variable
2. sufficiently constrain each computed witness variable
    1. here sufficiently means there's exactly 1 solution that satisfies the constraint.

In the sha256 circuit, the most tricky part boils down to these expressions:
1. $s_0 = (n\gg7|n\ll25) \oplus (n\gg18|n\ll14) \oplus (n\gg3)$
    1. $n \in [0, 2^{32})$, and already constrained.
2. $s_1 = (n\gg17|n\ll15) \oplus (n\gg19|n\ll13) \oplus (n\gg10)$
    1. $n \in [0, 2^{32})$, and already constrained.
3. $ch = (e \land f) \oplus ((!e) \land g)$
    1. $e,f,g \in [0, 2^{32})$, and already constrained.
4. $maj = (a \land b) \oplus (a \land c) \oplus (b \land c)$
    1. $a,b,c \in [0, 2^{32})$, and already constrained.

Notations:
1. $\oplus$ denotes bitwise xor.
2. $\land$ denotes bitwise and.
3. $|$ denotes bitwise or.
4. $\gg$ denotes right shift.
5. $\ll$ denotes left shift.

The arithmetization for the above expressions heavily applies properties of a concept called "spread integer".

Put simply, "spread integer" pads a 0 bit in front of each bit of the original value, i.e., $spread(11) = 0101, spread(1001) = 01000001$ etc. More specificly, the circuit will use 16 bit spread table where each original value is 16 bit, with a total of $2^{16}$ rows.

The properties of "spread integer" include:
1. $\forall a,b,c \in [0, 2^{32})$:
    1. $a \oplus b = EvenBits(spread(a) + spread(b))$, we'll refer to this as `P1`.
    2. $a \oplus b \oplus c = EvenBits(spread(a) + spread(b) + spread(c))$, we'll refer to this as `P2`.
    3. $(a \land b) \oplus (b \land c) \oplus (a \land c) = OddBits(spread(a) + spread(b) + spread(c))$, we'll refer to this as `P5`.
        1. This may not look so obvious at the 1st glance, unless you notice this: the i-th bit of the result is `1` iff at least two bits of $A_i/B_i/C_i$ is `1`.
    4. $(a \land b) \oplus ((!a) \land c) = OddBits(spread(a) + spread(b)) + OddBits(spread(!a) + spread(c))$, we'll refer to this as `P6`.
        1. This may not look so obvious at the 1st glance, unless you notice this: the i-th bit of $OddBits(spread(a) + spread(b))$ and $OddBits(spread(!a) + spread(c))$ can not be both `1`, because the i-th bit of $a$ and $!a$ cant not be both `1`.
2. $\forall a,b \in [0, 2^{32}), a \land b = OddBits(spread(a) + spread(b))$, we'll refer to this as `P3`.
3. $\forall v \in \mathbb{Z^{+}}, v = spread(EvenBits(v)) + 2 spread(OddBits(v))$, we'll refer to this as `P4`.

Notations:
1. EvenBits(v) denotes the integer composed of the even bits of the integer v, where the least significant bit is considered the 0th bit, thus an even bit, i.e., $EvenBits(1011_b) = 01_b = 1_b, EvenBits(1100_b) = 10_b$.
2. OddBits(v) denotes the integer composed of the odd bits of the integer v, i.e., $OddBits(1011_b) = 11_b, OddBits(1100_b) = 10_b$.

Also note that in the plonkish circuit, there're only limited constraining toolkits we can use:
1. circuit constraint, in the form of $q_0 w_0 + q_1 w_1 + ... q_n w_n + q_m w_0 w_1 + q_c = 0$
2. lookup constraint
3. permutation constraint

That's why we need to employ tricks to constrain the above expressions.

With these in mind, let's dive into the details.

# Arithmetization

0. prepare a 16 bit spread table that constrains this mapping: $\forall v \in [0, 2^{16})$ -> spread(v).
1. $s_0 = (n\gg7|n\ll25) \oplus (n\gg18|n\ll14) \oplus (n\gg3)$
    1. Here n is 32 bit, if we decompose it into 14-11-4-3 bit parts as (d, c, b, a), then $s_0 = (b||a||d||c)  \oplus (c||b||a||d)\oplus(0^{[3]}||d||c||b)$, thus we can apply the above spread integer property:
        1. Constrain $a,b,c,d$ as follows:
            1. Lookup $a,b,c,d$ in a 16 bit spread table, resulting in $a',b',c',d'$
                1. This ensures $a,b,c,d \in [0, 2^{16})$
            2. Further constrain $a,b,c,d$ by: $n = d + 2^{3}c + 2^{7}b + 2^{18}d$
            3. The above two steps **sufficiently** constrains:
                1. (d, c, b, a) to be the correct decomposation of n.
                2. $a',b',c',d'$ are the corresponding spread form of $a,b,c,d$.
    2. According to `P2`, $s_0 = EvenBits(spread((b||a||d||c)) + spread((c||b||a||d)) + spread((0^{[3]}||d||c||b))) = EvenBits((b'||a'||d'||c') + (c'||b'||a'||d') + (0^{[6]}||d'||c'||b')) = EvenBits((4^{28}b'+4^{25}a'+4^{11}d'+c') + (4^{21}c' + 4^{17}b'+4^{14}a'+d') + (4^{15}d'+4^4c'+b')) = EvenBits((4^{25}+4^{14})a' + (4^{28}+4^{17}+1)b' + (4^{21}+4^4+1)c' + (4^{15}+4^{11}+1)d')$
    3. Thus we can allocate a new witness variable R' and constrain it **sufficiently** with $R' = (4^{25}+4^{14})a' + (4^{28}+4^{17}+1)b' + (4^{21}+4^4+1)c' + (4^{15}+4^{11}+1)d'$
    4. It's not too hard to see that $R' \lt 2^{64}$, as it's a sum of 3 64-bit integers with the most significant bit being 0.
    5. Decompose R' into 2 32-bit parts $(R'^H, R'^L)$, s.t. $R' = R'^L + 2^{32} R'^H$, apply `P4`, thus $R' = spread(EvenBits(R'^L)) + 2 spread(OddBits(R'^L)) + 2^{32} (spread(EvenBits(R'^H)) + 2 spread(OddBits(R'^H)))$
    6. Thus we can lookup $EvenBits(R'^L)/OddBits(R'^L)/EvenBits(R'^H)/OddBits(R'^H)$ in the spread table, resulting in $r_1/r_2/r_3/r_4$, then further constrain it by $R' = r_1 + 2 r_2 + 2^{32} r_3 + 2^{33} r_4$
    7. We claim $EvenBits(R'^L)/OddBits(R'^L)/EvenBits(R'^H)/OddBits(R'^H)$ are **sufficiently** constrained.
        1. Because every different assignment of $EvenBits(R'^H)/OddBits(R'^H)$ will generate a different $R'^H$, every different assignment of $EvenBits(R'^L)/OddBits(R'^L)$ will generate a different $R'^L$, so there can't be another assignment of $EvenBits(R'^L)/OddBits(R'^L)/EvenBits(R'^H)/OddBits(R'^H)$ that can generate the same R'.
    8. $s_0 = EvenBits(R'^L) + 2^{16}EvenBits(R'^H)$
    9. $\mathbb{qed}$
2. $s_1 = (n\gg17|n\ll15) \oplus (n\gg19|n\ll13) \oplus (n\gg10)$
    1. Use similar tricks as $s_0$.
    2. $\mathbb{qed}$
3. $ch = (e \land f) \oplus ((!e) \land g)$
    1. Lookup $e/f/g$ in the spread table to get **sufficiently** constrained $e'/f'/g'$
        1. Needs to decompose into 16 bit parts first.
    2. Compute $p' = e' + f', q'=spread(2^{32}-1) - e' + g'$,
    3. Decompose $p'$ into **sufficiently** constrained even and odd bits $p'^{odd}/p'^{even}$ using similar tricks as $s_0$. Similar for $q'$.
    4. According to `P6`, $ch = p'^{odd} + q'^{odd}$
    5. $\mathbb{qed}$
4. $maj = (a \land b) \oplus (a \land c) \oplus (b \land c)$
    1. Lookup $a/b/c$ in the spread table to get **sufficiently** constrained $a'/b'/c'$
        1. Needs to decompose into 16 bit parts first.
    2. **Sufficiently** constrain $OddBits(a' + b' + c')$ using similar tricks as $s_0$.
    3. According to `P5`, $maj = OddBits(a' + b' + c')$
    4. $\mathbb{qed}$