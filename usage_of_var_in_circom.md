# Introduction

`circom` is one of the most frequently used programing languages for zk dapps.

There're two important concepts in it: `signal` and `var`.

`signal` can be regarded as variables in zkp, each signal can only be assigned once.

What about `var`?

Let's dive into it through an example.

# `var` by example

```circom
// Similar to Decoder template from circomlib/circuits/multiplexer.circom
// Returns a bit array `out` with a 1 at index `index`, and 0s everywhere else
template SingleOneArray(len) {
    signal input index;

    signal output out[len];
    signal success;
    var lc = 0;

    for (var i = 0; i < len; i++) {
        out[i] <-- (index == i) ? 1 : 0;
        out[i] * (index-i) === 0;
        lc = lc + out[i];
    }
    lc ==> success;
    // support array sizes up to a million. Being conservative here b/c according to Michael this template is very cheap
    signal should_be_all_zeros <== GreaterEqThan(20)([index, len]);
    success === 1 * (1 - should_be_all_zeros);
}
```
([source](https://github.com/aptos-labs/keyless-zk-proofs/blob/fd160220a88a5becf0f91ea1a5425fdd537c7399/circuit/templates/helpers/arrays.circom#L106-L125))

In the above code, `out[i] * (index-i) === 0;` constrains that `out[i]` is `0` for all indices except `index`.

But initially I didn't see how it constrains that `out[index]` is `0`.

My reasoning was as follows:

> There're only constraints between `success` and `lc`(as in `lc ==> success;`), but not between `lc` and `out`(since it's using `=` in `lc = lc + out[i]`, which is only assignment).

So what's actually happening here?

# Conclusion

In the end I got the answer from the official team:

> Circom compiler substitutes var in a constraint expression with the expression assigned to the var.

This means when `var` occurs in the constraint expression, it's actually constraining the signals in the expression of the `var`.