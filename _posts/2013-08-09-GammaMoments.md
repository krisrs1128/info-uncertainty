---
layout: post
title: "Gamma Moments"
description: "Relating gamma variables to calculate moments."
tags: [Change of Variables, Basic Probability, Gamma Distribution]
---

Let $X \sim \text{Gamma}(\alpha, \lambda)$, that is, distributed with density
$$
f_{X}(x) = \frac{\lambda^{\alpha}}{\Gamma\left(\alpha\right)}x^{\alpha - 1}e^{-\lambda x}.
$$
In this post, I'll give one approach to calculating $E\left(X^{n}\right)$. The problem can be solved more directly than I do here,  but I think one of the ideas behind this approach -- relating a Gamma$\left(\alpha, \lambda\right)$ variable to a simpler Gamma$\left(\alpha, 1\right)$ variable -- is generally useful in other contexts, which is why I find it worth going through the extra effort here.

The overall approach is to first show that if $Y = \lambda X$ then, $Y \sim \text{Gamma}\left(\alpha,  1\right)$. Then, the moments of $Y$ are easy to compute, and the moments of $X$ then follow immediately.

**Claim** *If $X \sim \text{Gamma}(\alpha, \lambda)$ then $Y = \lambda X \sim \text{Gamma}\left(\alpha,  1\right)$.*

**Pf.** This is a simple change of variables. The inverse mapping is $X = \frac{1}{\lambda}Y$, which has Jacobian $\frac{1}{\lambda}$, so the density of $Y$ is

$$
\begin{align}
f\_{Y}\left(y\right) &= f\_{X}\left(\frac{1}{\lambda}y\right) \left|\frac{1}{\lambda}\right| \\\
&= \frac{\lambda^{\alpha}}{\Gamma\left(\alpha\right)}\left(\frac{y}{\lambda}\right)^{\alpha - 1}e^{- \lambda \frac{y}{\lambda}} \frac{1}{\lambda} \\\
&= \frac{1}{\Gamma\left(\alpha\right)}y^{\alpha - 1}e^{-y}
\end{align},
$$
which is the density of a Gamma$\left(\alpha, 1\right)$ variable, as needed.

**Claim** *If $Y \sim \Gamma\left(\alpha, 1\right)$, then $E\left(Y^{n}\right) = \frac{\Gamma\left(\alpha + n\right)}{\Gamma\left(\alpha\right)}$.*

**Pf.**. Observe that
$$
\begin{align}
E\left(Y^{n}\right) &= \int\_{0}^{\infty} \frac{1}{\Gamma\left(\alpha\right)}y^{n}y^{\alpha - 1}e^{-y} dy
\\\
&= \frac{\Gamma\left(\alpha + n \right)}{\Gamma\left(\alpha \right)} \int_{0}^{\infty} \frac{1}{\Gamma\left(\alpha + n\right)} y^{\alpha + n - 1}e^{-y} dy \\\
&= \frac{\Gamma\left(\alpha + n\right)}{\Gamma\left(\alpha\right)},
\end{align}
$$
where, for the last equality, we recognized the integrand as the Gamma$\left(\alpha + n, 1\right)$ density.

We now can easily compute $E\left(X^{n}\right)$, our original goal. Using the previous scaling $X = \frac{1}{\lambda}Y$, we have
$$
\begin{align}
E\left(X^{n}\right) &= E\left(\frac{Y^{n}}{\lambda^{n}}\right) \\\
&= \frac{1}{\lambda^{n}}\frac{\Gamma\left(\alpha + n\right)}{\Gamma\left(\alpha\right)} \\\
&= \frac{\alpha\left(\alpha + 1\right) \dots \left(\alpha + n - 1\right)}{\lambda^{n}},
\end{align}
$$
where for the last inequality we used the fact that $\Gamma\left(k\right) = \left(k - 1\right)!$ for integers $k$. This is the desired $n^{th}$ moment of a Gamma$\left(\alpha, \lambda\right)$ variable.