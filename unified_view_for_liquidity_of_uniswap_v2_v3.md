# Introduction

We know uniswap v2 is based on the constant-product formula:

$$
x * y = l^2
$$

where `x` means `token0` amount, `y` means `token1` amount, `l` means liquidity.

We also know uniswap v3 is much more complex([FYI](https://app.uniswap.org/whitepaper-v3.pdf)) than v2.

But in this article, we'll look at the liquidity of uniswap v2 and v3 in a unified view.


# Definition of liquidity

The key for the unified view is the unified definition of liquidity for both uniswap v2 and v3:

Providing liquidity `l` means the provided `token0`/`token1` should cover the possible decrease(negative delta) of `token0`/`token1` on the curve $x * y = l^2$ when the price `p` defined as `p=y/x` transitions *continously* from the current price to all possible values in range.

For v2, the possible price range is (0, +∞).

For v3, the possible price range is specified by user, e.g., $[price_{low}, price_{high}]$.

# Liquidity of v2

When the price transitions from `p` towards 0, `token1` decreases from $\sqrt{l \times p}$ to 0. So liquidity provider needs to provide $\sqrt{l \times p}$ of `token1`.

When the price transitions from `p` towards +∞, `token0` decreases from $\sqrt{\frac{l}{p}}$ to 0. So liquidity provider needs to provide $\sqrt{\frac{l}{p}}$ of `token0`.

In total, needs to provide $\sqrt{\frac{l}{p}}$ of `token0` and $\sqrt{l \times p}$ of `token1`.

# Liquidity of v3

There're 3 cases:

## If `p` < $price_{low}$

When the price transitions from $price_{low}$ to $price_{high}$, `token0` decreases from $\sqrt{\frac{l}{price_{low}}}$ to $\sqrt{\frac{l}{price_{high}}}$, so liquidity provider needs to provide $\sqrt{\frac{l}{price_{low}}} - \sqrt{\frac{l}{price_{high}}}$ of `token0`.

([src](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L328-L335))

## If $price_{low}$ <= `p` <= $price_{high}$

When the price transitions from `p` to $price_{low}$, `token1` decreases from $\sqrt{l \times p}$ to $\sqrt{l \times price_{low}}$, so liquidity provider needs to provide $\sqrt{l \times p} - \sqrt{l \times price_{low}}$ of `token1`.

When the price transitions from `p` to $price_{high}$, `token0` decreases from $\sqrt{\frac{l}{p}}$ to $\sqrt{\frac{l}{price_{high}}}$, so liquidity provider needs to provide $\sqrt{\frac{l}{p}} - \sqrt{\frac{l}{price_{high}}}$ of `token0`.

In total, needs to provide $\sqrt{\frac{l}{p}} - \sqrt{\frac{l}{price_{high}}}$ of `token0` and $\sqrt{l \times p} - \sqrt{l \times price_{low}}$ of `token1`.

([src](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L350-L359))

## If `p` > $price_{high}$

When the price transitions from $price_{high}$ to $price_{low}$, `token1` decreases from $\sqrt{l \times price_{high}}$ to $\sqrt{l \times price_{low}}$, so liquidity provider needs to provide $\sqrt{l \times price_{high}} - \sqrt{l \times price_{low}}$ of `token1`.

([src](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/UniswapV3Pool.sol#L365-L369))