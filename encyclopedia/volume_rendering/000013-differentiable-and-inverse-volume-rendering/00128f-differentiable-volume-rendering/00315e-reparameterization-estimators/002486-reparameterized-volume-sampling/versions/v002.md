# Reparameterized Volume Sampling

Also called: Differentiable Volume Sampling, Inverse CDF Sampling

## Overview
Reparameterized Volume Sampling is a differentiable rendering technique that computes color estimates by sampling points on a ray using the inverse of the cumulative distribution function (CDF) of the density field [S1, unofficial]. It solves the problem of computing gradients with respect to the density field parameters by making the sampling locations differentiable functions of the density, allowing gradients to back-propagate through the sample positions [S1, unofficial]. Unlike NeRF's hierarchical sampling which uses an auxiliary coarse model trained separately, this method learns sampling points end-to-end without auxiliary networks [S1, unofficial]. Compared to standard importance sampling with uniform distribution, this stratified reparameterized sampling exhibits lower variance [S1, unofficial].

## How it works
**Inputs:**
- Ray $\mathbf{r}$ with near/far bounds $[t_n, t_f]$.
- Density field $\sigma_{\mathbf{r}}(t)$ parameterized by $\theta$.
- Radiance field $c_{\mathbf{r}}(t)$ parameterized by $\theta$.
- Number of samples per ray $k$ (typically 32) [S1, unofficial].
- Number of knots $m$ for piecewise linear approximation (typically 256) [S1, unofficial].

**Outputs:**
- Color estimate $C(\mathbf{r})$.
- Gradients $\nabla_\theta C(\mathbf{r})$.

**Data Structures:**
- **Knots:** A set of $m+1$ points $t_1, \dots, t_{m+1}$ along the ray, typically a perturbed uniform grid [S1, unofficial].
- **Density Values:** $\sigma_i = \sigma_{\mathbf{r}}(t_i)$ for $i=1 \dots m+1$.
- **Cumulative Integrals:** Precomputed integrals of the piecewise linear density approximation between knots.

**Method Steps:**

1.  **Approximate Density:** Approximate the density field $\sigma_{\mathbf{r}}(t)$ along the ray using a piecewise linear interpolation between knots $t_i$ and $t_{i+1}$:
    $$ \sigma_{\mathbf{r}}(t) \approx \sigma_i + \frac{\sigma_{i+1} - \sigma_i}{t_{i+1} - t_i} (t - t_i) \quad \text{for } t \in [t_i, t_{i+1}] $$
    [S1, unofficial]

2.  **Compute CDF:** The CDF is defined as:
    $$ F_{\mathbf{r}}(t) = 1 - \exp\left(-\int_{t_n}^t \sigma_{\mathbf{r}}(s) ds\right) $$
    [S1, unofficial]
    Due to the piecewise linear density, the integral $\int_{t_n}^t \sigma_{\mathbf{r}}(s) ds$ is a piecewise quadratic function. Let $I(t)$ denote this integral approximation.

3.  **Generate Noise:** Generate $k$ uniform random samples $u_j \in [0, 1]$ for $j=1 \dots k$.

4.  **Target Opacity:** Compute the target cumulative opacity value for stratified sampling or individual samples:
    $$ y_j = F_{\mathbf{r}}(t_f) \cdot u_j $$
    [S1, unofficial]

5.  **Rewrite Inversion:** Rewrite the inversion equation $y_j = F_{\mathbf{r}}(t)$ as:
    $$ -\log(1 - y_j) = \int_{t_n}^t \sigma_{\mathbf{r}}(s) ds = I(t) $$
    [S1, unofficial]
    Let $Y_j = -\log(1 - y_j)$. The goal is to solve $I(t) = Y_j$ for $t$.

6.  **Find Bin:** Evaluate the cumulative integral $I(t_i)$ at all knots $t_i$. Find the interval $[t_i, t_{i+1}]$ such that $I(t_i) \le Y_j < I(t_{i+1})$. This identifies the bin containing the solution $t$.

7.  **Solve for $t$:** Within the identified bin, $I(t)$ is a quadratic function of $t$. Solve the quadratic equation $I(t) = Y_j$ for $t$ within $[t_i, t_{i+1}]$. The explicit formulae for $t$ are provided in the supplementary materials of the source [S1, unofficial]. The solution $t$ is a differentiable function of the density field $\sigma_{\mathbf{r}}$.

8.  **Evaluate Radiance:** Evaluate the radiance $c_{\mathbf{r}}(t)$ at the computed sample position $t$.

9.  **Accumulate Color:** Accumulate the color estimate using the standard volume rendering equation with the computed samples.

**Parameters:**
- Number of samples per ray $k$: 32 [S1, unofficial].
- Number of knots $m$: 256 [S1, unofficial].
- Activation function for density field: Softplus with $\beta = 10$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **ReLU Activation:** Using ReLU activation for the density field can result in zero gradients, hindering training [S1, unofficial]. Softplus is required to ensure differentiability [S1, unofficial].
- **Numerical Stability:** The source notes that explicit formulae for $t$ and solutions crucial for numerical stability are provided in the supplementary materials [S1, unofficial]. Implementers must handle the quadratic solution carefully to avoid numerical errors.
- **Piecewise Constant Approximation:** While faster, piecewise constant approximation delivered worse rendering quality compared to piecewise linear [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The method requires evaluating the density field at $m$ knots and solving quadratic equations for each sample. The complexity is dominated by the density evaluation and the root-finding step.
- **Memory Complexity:** Requires storing density values at $m$ knots and precomputed integrals.
- **Performance:** In the 'foggy' density field test, 32 samples matched the baseline performance of 128 samples [S1, unofficial]. In the 'Glass and wall' and 'Wall' density fields, 4 samples matched the baseline performance [S1, unofficial].
- **Variance:** Stratified reparameterized sampling exhibits lower variance compared to naive importance sampling with uniform distribution [S1, unofficial].
- **Strengths:** End-to-end differentiable sampling without auxiliary networks; lower variance than standard importance sampling; efficient with fewer samples for certain scenes.
- **Limitations:** Requires careful implementation of the quadratic root-finding for numerical stability; piecewise constant approximation yields worse quality.

## Background
Reparameterized Volume Sampling was introduced in the paper "Differentiable Rendering With Reparameterized Volume Sampling" [S1, unofficial]. It builds on the reparameterization trick [S1, unofficial]. The method was applied to 3D scene reconstruction based on a set of image projections, using synthetic data from NeRF [S1, unofficial]. It was compared against NeRF with and without hierarchical sampling, and against the original DVGO [S1, unofficial].

## Sources
[S1] Differentiable Rendering With Reparameterized Volume Sampling, https://openreview.net/pdf?id=zHSaBQtj-l, fetched 2026-10-29 (unofficial)
