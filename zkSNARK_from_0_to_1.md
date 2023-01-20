# Introduction

zkSNARK is short for zero knowledge succinct non-interactive arguments of knowledge, the truly ingenious method of non-interactively and succinctly proving that something is true without revealing any other information. It's kinda of a "signature" for computation, and mainly used for privacy computation and verifiable computation.

# Protocol

1. Setup
	1. select a generator g and a cryptographic pairing e
	2. suppose the domain problem has already been converted into arithmetic circuit via arithmetization, for a function f(u) = y with with m input/output variables, convert arithmetic circuit into QAP form({$l_{i}(x)$, $r_{i}(x)$, $o_{i}(x)$}$_{i\in\{0,...,n\}}$, t(x)) of degree d and size n + 1.
	3. sample random s, $ρ_l$, $ρ_r$, $α_l$, $α_r$, $α_o$, β, γ
	4. set $ρ_o = ρ_l * ρ_r$ and the operand generators $g_l = g^{p_l}$, $g_r = g^{p_r}$, $g_o = g^{p_o}$
	5. set the proving key:
		$$\left( \{g^{s^k}\}_{k\in[d]}, \{g_l^{l_i(s)}, g_r^{r_i(s)}, g_o^{o_i(s)}\}_{i\in\{0,...,n\}}, \\
		\{g_l^{α_{l}l_i(s)}, g_r^{α_{r}r_i(s)}, g_o^{α_{o}o_i(s)}, g_l^{βl_i(s)}g_r^{βr_i(s)}g_o^{βo_i(s)}\}_{i\in\{m+1,...,n\}}, \\
		g_l^{t(s)}, g_r^{t(s)}, g_o^{t(s)}, g_l^{α_{l}t(s)}, g_r^{α_{r}t(s)}, g_o^{α_{o}t(s)}, g_l^{βt(s)}, g_r^{βt(s)}, g_o^{βt(s)}  \right)$$
	6. set the verification key:
		$$\left( g^1, g_o^t(s), \{g_l^{l_i(s)}, g_r^{r_i(s)}, g_o^{o_i(s)} \}_{i\in\{0,...,m\}}, g^{α_{l}}, g^{α_{r}}, g^{α_{o}}, g^γ, g^{βγ} \right)$$
2. Proving
	1. for the input u, execute the computation of f(u) obtaining values $\{v_i\}_{i\in\{m+1,...,n\}}$ for all the itermediary variables
	2. assign all values to the unencrypted variable polynomials $L(x)=l_0(x) + \sum_{i=1}^{n} v_i*l_i(x)$, and similarly R(x), O(x)
	3. sample random $δ_l, δ_r, δ_o$
	4. find $h(x) = \frac{L(x)R(x)-O(x)}{t(x)} + δ_rL(x) + δ_lR(x) + δ_lδ_rt(x) - δ_o$
	5. assign the prover’s variable values to the encrypted variable polynomials and apply zero-knowledge δ-shift $g_l^{L_p(s)} = \left(g_l^{t(s)}\right)^{δ_l} * \prod_{i=m+1}^{n} \left(g_l^{l_i(s)}\right)^{v_i}$ and similarly $g_r^{R_p(s)}, g_o^{O_p(s)}$
	6. assign its α-shifted pairs $g_l^{L_p'(s)} = \left( g_l^{α_{l}t(s)}\right)^{δ_l} * \prod_{i=m+1}^{n} \left(g_l^{α_ll_i(s)}\right)^{v_i}$
	7. assign the variable values consistency polynomials $$g^{Z(s)} = \left(g_l^{βt(s)}\right)^{δ_l} \left(g_r^{βt(s)}\right)^{δ_r} \left(g_o^{βt(s)}\right)^{δ_o} * \prod_{i=m+1}^{n} \left( g_l^{βl_i(s)} g_r^{βr_i(s)} g_o^{βo_i(s)} \right)^{v_i} $$
	8. compute the proof $\left( g_l^{L_p(s)}, g_r^{R_p(s)}, g_o^{O_p(s)}, g^{h(s)}, g_l^{L'_p(s)}, g_r^{R'_p(s)}, g_o^{O'_p(s)}, g^{Z(s)} \right)$

3. Verification
	1. parse a provided proof as $\left( g_l^{L_p}, g_r^{R_p}, g_o^{O_p}, g^h, g_l^{L'_p}, g_r^{R'_p}, g_o^{O'_p}, g^{Z} \right)$
	2. assign input/output values to verifier’s encrypted polynomials and add to 1:  
		$ g_l^{L_v(s)} = g_l^{l_0(s)} * \prod_{i=1}^{m} \left(g_l^{l_i(s)}\right)^{v_i} $ and similarly for $g_r^{R_v(s)}$ and $g_o^{O_v(s)}$
	3. variable polynomials restriction check :  
		$e\left(g_l^{L_p}, g^{α_{l}} \right) = e\left(g_l^{L_p'}, g \right)$ and similarly for $g_r^{R_p}$ and $g_o^{O_p}$
	4. variable values consistency check:  
		$e\left(g_l^{L_p} g_r^{R_p} g_o^{O_p}, g^{βγ}\right) = e\left(g^Z, g^γ\right)$
	5. valid operations check:  
		$e\left(g_l^{L_p}g_l^{L_v}, g_r^{R_p}g_r^{R_v}\right) = e\left(g_o^{t(s), g^h}\right) e\left(g_o^{O_p}g_o^{O_v}, g\right)$

# Principle

After converting into QAP form, the circuit is equivalent to constrains in this form: 
	$$ \begin{equation}
			\begin{cases}
			\sum_{i=0}^{i=n}l_{0i}x_{i}*\sum_{i=0}^{i=n}r_{0i}x_{i}=\sum_{i=0}^{i=n}o_{0i}x_{i}\\
			\sum_{i=0}^{i=n}l_{1i}x_{i}*\sum_{i=0}^{i=n}r_{1i}x_{i}=\sum_{i=0}^{i=n}o_{1i}x_{i}\\
			... \\
			\sum_{i=0}^{i=n}l_{di}x_{i}*\sum_{i=0}^{i=n}r_{di}x_{i}=\sum_{i=0}^{i=n}o_{di}x_{i}\\
			\end{cases}
		\end{equation} $$

Each equation corresponds to a gate. We can convert the equation system to a single polynomial equation if we assign a different value $a_j$ to gate:
	$$ \sum_{i=0}^{i=n}l_{i}(x)x_{i}*\sum_{i=0}^{i=n}r_{i}(x)x_{i}=\sum_{i=0}^{i=n}o_{i}(x)x_{i} $$

 where $$\{l_i(a_j)=l_{ji}, r_i(a_j)=r_{ji}, o_i(a_j)=o_{ji}\}_{j\in [d], i\in [n]}$$

Let $$t(x)=\prod_{j=0}^{j=d}(x-a_j)\\
	l(x)=\sum_{i=0}^{i=n}l_{i}(x)x_{i}\\
	r(x)=\sum_{i=0}^{i=n}r_{i}(x)x_{i}\\
	o(x)=\sum_{i=0}^{i=n}o_{i}(x)x_{i}$$ the equation system holds iff $t(x)$ divides $l(x)*r(x)-o(x)$

In other words, the equation system holds iff : $$l(x)*r(x)-o(x)=t(x)*h(x)$$

where $h(x)$ is also a polynomial.

In elliptic curve addition is easier to calculate than subtraction, so we can convert the above equation to: $$l(x)*r(x)=t(x)*h(x)+o(x)$$

So the equation system is equivalent to $l(x)*r(x)=t(x)*h(x)+o(x)$ under the precondition that 
	$$ \begin{equation}
		\begin{cases}
		l(x)=\sum_{i=0}^{i=n}l_{i}(x)x_{i}\\
		r(x)=\sum_{i=0}^{i=n}r_{i}(x)x_{i}\\
		o(x)=\sum_{i=0}^{i=n}o_{i}(x)x_{i}\\
		\end{cases}
	\end{equation} $$ for some assignment of $\{x_i\}_{i\in[d]}$

This is where "knowledge of exponent assumption"(KEA for short) plays its role in the protocol.

The KEA states that given pairs $\{(g^{c_i}, g^{c_iα})\}_{i\in[n]}$ for some random $α$, the only way to generate $(g^k, g^{kα})$ is: pick $\{k_i\}_{i\in[n]}$, and compute $\{g^{c_ik_i}=(g^{c_i})^{k_i}, g^{c_iαk_i}=(g^{c_iα})^{k_i}=(g^{c_ik_i})^{α}\}_{i\in[n]}$, thus $\prod_{i=0}^{n}g^{c_ik_i}=\prod_{i=0}^{n}(g^{c_i})^{k_i}$ and $\prod_{i=0}^{n}g^{c_iαk_i}=\prod_{i=0}^{n}(g^{c_ik_i})^{α}=(\prod_{i=0}^{n}(g^{c_ik_i}))^{α}$.

So KEA can be used to constrain $l(x)$/$r(x)$/$o(x)$ to be in its desired form separately, but $x_i$ appears not only in $l(x)$ but also in $r(x)$ and $o(x)$, so we also need another extension trick of KEA to ensure cross consistency:   

Given $\{(g^{l_i}, g^{r_i}, g^{o_i}, g^{l_iβ_l+r_iβ_r+o_iβ_o})\}_{i\in[n]}$ for some random $β_l$, $β_r$, $β_o$, the only way to generate $(g^l, g^r, g^o, g^{lβ_l+rβ_r+oβ_o})$ is: pick $\{x_i\}_{i\in[n]}$, and compute $\{g^{l_ix_i}=(g^{l_i})^{x_i}, g^{r_ix_i}=(g^{r_i})^{x_i}, g^{o_ix_i}=(g^{o_i})^{x_i}, g^{l_ix_iβ_l+r_ix_iβ_r+o_ix_iβ_o}=(g^{l_iβ_l+r_iβ_r+o_iβ_o})^{x_i}=(g^{l_ix_i})^{β_l}(g^{r_ix_i})^{β_r}(g^{o_ix_i})^{β_o}\}_{i\in[n]}$, thus $\prod_{i=0}^{n}g^{l_ix_iβ_l+r_ix_iβ_r+o_ix_iβ_o}=\prod_{i=0}^{n}(g^{l_ix_i})^{β_l}(g^{r_ix_i})^{β_r}(g^{o_ix_i})^{β_o}=(\prod_{i=0}^{n}(g^{l_ix_i}))^{β_l}(\prod_{i=0}^{n}(g^{r_ix_i}))^{β_r}(\prod_{i=0}^{n}(g^{o_ix_i}))^{β_o}$

Now we're ready to come up with the essential part of the protocol, in summary:  
1. use random s to constrain $l(x)*r(x)-o(x)=t(x)*h(x)$
2. use KEA to constrain $l(s)$/$r(s)$/$o(s)$ to be in desired form
3. use the extension trick of KEA to ensure cross consistency

In detail:
1. Setup
	1. sample random s, $α_l$, $α_r$, $α_o$, $β_l$, $β_r$, $β_o$
	2. set the proving key: 
		$$\left( \{g^{s^k}\}_{k\in[d]}, \{g^{l_i(s)}, g^{r_i(s)}, g^{o_i(s)}\}_{i\in\{0,...,n\}}, \\
		\{g^{α_{l}l_i(s)}, g^{α_{r}r_i(s)}, g^{α_{o}o_i(s)}, g^{β_ll_i(s)}g^{β_rr_i(s)}g^{β_oo_i(s)}\}_{i\in\{0,...,n\}}, \\
		\right)$$
	3. set the verification key:
		$$\left( g^t(s), g^{α_l}, g^{α_r}, g^{α_o}, g^{β_l}, g^{β_r}, g^{β_o}\right)$$
2. Proving:
	1. for the input u, execute the computation of f(u) obtaining values $\{v_i\}_{i\in\{m+1,...,n\}}$ for all the itermediary variables
	2. assign all values to the unencrypted variable polynomials $L(x)=l_0(x) + \sum_{i=1}^{n} v_i*l_i(x)$, and similarly R(x), O(x)
	3. find $h(x) = \frac{L(x)R(x)-O(x)}{t(x)}$
	4. assign the prover’s variable values to the encrypted variable polynomials and apply zero-knowledge δ-shift $g^{L(s)} = \prod_{i=0}^{n} \left(g_l^{l_i(s)}\right)^{v_i}$ and similarly $g^{R(s)}, g^{O(s)}$
	5. assign its α-shifted pairs $g^{L'(s)} = \prod_{i=0}^{n} \left(g^{α_ll_i(s)}\right)^{v_i}$ and similarly $g^{R'(s)}, g^{O'(s)}$
	6. assign the variable values consistency polynomials $$g^{Z(s)} = \prod_{i=0}^{n} \left( g^{β_ll_i(s)} g^{β_rr_i(s)} g^{β_oo_i(s)} \right)^{v_i} $$
	7. compute the proof $\left( g^{L(s)}, g^{R(s)}, g^{O(s)}, g^{h(s)}, g^{L'(s)}, g^{R'(s)}, g^{O'(s)}, g^{Z(s)} \right)$
3. Verification
	1. variable polynomials restriction check:  
		$e(g^L,g^{α_l}) = e(g^{L'}, g), e(g^R,g^{α_r}) = e(g^{R'}, g), e(g^O,g^{α_o}) = e(g^{O'}, g)$
	2. variable values consistency check:  
		$e(g^L, g^{β_l})e(g^R, g^{β_r})e(g^O, g^{β_o}) = e(g^Z, g)$
	3. valid operations check:  
		$e(g^L, g^R) = e(g^t, g^h)e(g^O, g)$

But there's a problem: the availability of $g^{β_l}, g^{β_r}, g^{β_o}$ allows to use different values of same variable in different operands. (Recall that in the extention trick of KEA only $g^{l_iβ_l+r_iβ_r+o_iβ_o}$ should be available.)

Luckily it's easy to fix:
1. Setup
	1. ... sample $β_l$, $β_r$, $β_o$, γ
	2. ... set verification key: $(..., g^{β_lγ}, g^{β_rγ}, g^{β_oγ}, g^γ)$
2. Proving ...
3. verification:  
	2. variable values consistency check: 
		$e(g^L, g^{β_lγ})e(g^R, g^{β_rγ})e(g^O, g^{β_oγ}) = e(g^Z, g^γ)$


We could also similarly mask the α-s to address the malleability of variable polynomials. However it is not necessary since any modification of a variable polynomial needs to be reflected in variable consistency polynomials which are not possible to modify.

It is important to note that we exclude the case when variable polynomials are all of 0-degree which otherwise would allow to expose encryptions of β in $g^{l_iβ_l+r_iβ_r+o_iβ_o}$ in case when any two is zero.

But the above protocol adds 4 expensive pairing operations, it can be optimized with a clever selection of the generators g for each operand ingraining the "shifts":  
1. Setup
	1. ... sample $β$, γ, $ρ_l$, $ρ_r$ and set $ρ_o=ρ_l*ρ_r$
	2. set generators $g_l=g^{ρ_l}, g_r=g^{ρ_r}, g_o=g^{ρ_o}$
	3. set proving key:
		$$\left( \{g^{s^k}\}_{k\in[d]}, \{g_l^{l_i(s)}, g_r^{r_i(s)}, g_o^{o_i(s)}, g_l^{α_ll_i(s)}, g_r^{α_rr_i(s)}, g_o^{α_oo_i(s)}, g_l^{βl_i(s)}, g_r^{βr_i(s)}, g_o^{βo_i(s)}\}_{i\in\{0,...,n\}} \right)$$
	4. set verification key: $\left( g_o^{t(s)}, g^{α_l}, g^{α_r}, g^{α_o}, g^{βγ}, g^γ\right)$
2. Proving
	1. ... assign variable values:
		$$g^{Z(s)} = \prod_{i=0}^{n} \left( g_l^{βl_i(s)} g_r^{βr_i(s)} g_o^{βo_i(s)} \right)^{v_i} $$
3. Verification
	1. variable polynomials restriction check:  
		$e(g_l^L,g^{α_l}) = e(g_l^{L'}, g)$, and similarly for $g_r^R, g_o^O$
	2. variable values consistency check:  
		$e(g_l^Lg_r^Rg_o^O, g^{βγ}) = e(g^Z, g^γ)$
	3. valid operations check:  
		$e(g_l^L, g_r^R) = e(g_o^t, g^h)e(g_o^O, g) => e(g, g)^{ρ_lρ_rLR} = e(g, g)^{ρ_lρ_rth+ρ_lρ_rO}$

(Note: there are further protocol improvements in the Jens Groth’s 2016 paper)

However the above protocol still has a flaw: it can't constrain constant assignment which is necessary for constraint like `a x 1 = b` where right operand has a constant in it.

Luckily public input comes to the rescure!

Using homomorphic encryption it's possible to split the variable polynomials between prover and verifier:
1. Setup
	1. ... separate all n+1 variable polynomials into two groups
		* verifier's m+1:  
			$L_v(x) = l_0(x) + l_1(x)x_1 + . . . + l_m(x)x_m$, and alike for $R_v(x)$ and $O_v(x)$, where index 0 is reserved for the value of $v_{one} = 1$
	2. set proving key:
		$$\left( \{g^{s^k}\}_{k\in[d]}, \{g_l^{l_i(s)}, g_r^{r_i(s)}, g_o^{o_i(s)}, g_l^{α_ll_i(s)}, g_r^{α_rr_i(s)}, g_o^{α_oo_i(s)}, g_l^{βl_i(s)}, g_r^{βr_i(s)}, g_o^{βo_i(s)}\}_{i\in\{m+1,...,n\}} \right)$$
	3. add to the verification key:
		$$\left( ..., \{g_l^{l_i(s)}, g_r^{r_i(s)}, g_o^{o_i(s)}\}_{i\in\{0,...m\}} \right)$$
2. Proving
	1. ... split the original L(x) into $L_p(x)$ and $L_v(x)$ such that $L(x)=L_p(x)+L_v(x)$, and similarly for R(x), O(x)
	2. provide the proof:
		$$\left( g_l^{L_p(s)}, g_r^{R_p(s)}, g_o^{O_p(s)}, g_l^{α_lL_p(s)}, g_r^{α_rR_p(s)}, g_o^{α_oO_p(s)}, g^{Z(s)}, g^{h(s)}   \right)$$
3. Verification
	1. assign verifier’s variable polynomial values:
		$$g_l^{L_v(s)}=g_l^{l_0(s)}\prod_{i=1}^{m}(g_l^{l_i(s)})^{x_i}$$
		and similarly for $g_r^{R_v(s)}, g_o^{O_v(s)}$
	2. variable polynomials restriction check:  
		$e\left(g_l^{L_p}, g^{α_l}\right)=e\left(g_l^{L_p'(s), g}\right)$ and similarly for $g_r^{R_p}, g_o^{O_p}$
	3. variable values consistency check:
		$$e\left(g_l^{L_p} g_r^{R_p} g_o^{O_p}, g^{βγ}\right)=e\left(g^Z, g^γ\right)$$
	4. valid operations check:
		$e\left(g_l^{L_p}g_l^{L_v}, g_r^{R_p}g_r^{R_v}\right) = e\left(g_o^{t(s)}, g^h\right) e\left(g_o^{O_p}g_o^{O_v}, g\right)$

	
We're only one step away from the protocol in the beginning: zero-knowledge.

As the construction has to accommodate the verifier’s input polynomials, we try **adding** randomness to the evaluations:

$$\require{cancel}
(L(s)+δ_l)(R(s)+δ_r) - (O(s)+δ_o)= t(s)(∆ + h(s))\\
	\cancel{L(s)R(s) - O(s)} + (δ_rL(s)+δ_lR(s)+δ_lδ_r-δ_o) = t(s)∆ + \cancel{t(s)h(s)}\\
	∆ = \frac{δ_rL(s)+δ_lR(s)+δ_lδ_r-δ_o}{t(s)}$$

Every term in the numerator is a multiple of $δ$, therefore we can make it divisible by multiplying
each $δ$ with $t(s)$:

$$\require{cancel}
(L(s)+δ_lt(s))(R(s)+δ_rt(s)) - (O(s)+δ_ot(s))= t(s)(∆ + h(s))\\
	\cancel{L(s)R(s) - O(s)} + t(s)(δ_rL(s)+δ_lR(s)+δ_lδ_r-δ_o) = t(s)∆ + \cancel{t(s)h(s)}\\
	∆ = δ_rL(s)+δ_lR(s)+δ_lδ_rt(s)-δ_o$$

Which we can efficiently compute in the encrypted space:

$$g_l^{L(s)+δ_lt(s)} = g_l^{L(s)}(g_l^{t(s)})^{δ_l}, etc\\
	g^∆ = (g^{L(s)})^{δ_r}(g^{R(s)})^{δ_l}(g^{t(s)})^{δ_lδ_r}g^{-δ_o}$$

This leads to passing of valid operations check while concealing the encrypted values.