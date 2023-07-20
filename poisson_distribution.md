# Introduction

The Poisson distribution is a discrete probability distribution that expresses the probability of a given number of events occurring in a fixed interval of time or space if these events occur with a known constant mean rate and independently of the time since the last event.

Most importantly, it has this property:

The number of events in any interval of length t is a Poisson random variable with parameter $\lambda t$.

From this alone, we can deduce its probability mass function:

$$Pr\lbrace N(t)=n\rbrace = \frac{(\lambda t)^n}{n!}e^{-\lambda t}$$

Which is amazing! 

But why? 

Let's dive into the details.

# Principle

A Poisson random variable $X$ with parameter $\lambda$ is a measurable function whose distribution function is:

$$Pr\lbrace X = k\rbrace = \lim_{N \to \infty}{N\choose k}P^k(1-P)^{N-k}$$

where $NP=\lambda$

Let's substitute $P=\frac{\lambda}{N}$ into the above equation:

$$
\begin{align*}
Pr\lbrace X = k\rbrace &= \lim_{N \to \infty}{N\choose k}({\frac{\lambda}{N}})^k(1-\frac{\lambda}{N})^{N-k} \\ 
            &= \frac{\lambda^k}{k!}\lim_{N \to \infty}\frac{N!}{(N-k)!N^k}(1-\frac{\lambda}{N})^N(1-\frac{\lambda}{N})^{-k}\\
            &= \frac{\lambda^k}{k!}\lim_{N \to \infty}\frac{N(N-1)...(N-k+1)}{N^k}(1-\frac{\lambda}{N})^{-k}(1-\frac{\lambda}{N})^N\\
            &= \frac{\lambda^k}{k!}\lim_{N \to \infty}(1-\frac{\lambda}{N})^N\\
            &= \frac{\lambda^k}{k!}\lim_{N \to \infty}((1-\frac{\lambda}{N})^{-\frac{N}{\lambda}})^{-\lambda}\\
            &= \frac{\lambda^k}{k!}\lim_{N \to \infty}e^{-\lambda}\\
            &= \frac{\lambda^k}{k!}e^{-\lambda}
\end{align*}
$$

Thus according to the property at the beginning:

$$Pr\lbrace N(t)=n\rbrace = \frac{(\lambda t)^n}{n!}e^{-\lambda t}$$

$\mathbb{qed}$