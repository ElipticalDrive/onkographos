# Free-Flight Based Gradient Estimators

Also called: DT estimators, delta tracking gradient estimators

## Overview
Free-flight based gradient estimators compute gradients of the volume rendering equation with respect to scene parameters by leveraging free-flight distance sampling, typically via delta tracking [S1]. They solve the problem of differentiating volumetric light transport in participating media, enabling inverse volume rendering and optimization [S1].

## How it works
**Inputs:**
*   Ray segment defined by origin, direction, and maximum distance $t_s$ (surface intersection or medium boundary) [S1].
*   Scene parameters $\theta$ (e.g., $\sigma_t$, emission coefficients) [S1].
*   Radiance quantities: $L_s(t)$ (in-scattered radiance), $L_e(t)$ (emitted radiance), $\alpha(t)$ (scattering albedo) [S1].

**Outputs:**
*   Gradient estimates $\langle \partial L \rangle$ with respect to $\theta$ [S1].

**Data Structures:**
*   Accumulator for transmittance gradients, initialized to zero [S1].
*   Sampled free-flight distance $t$ and list of null-scattering interaction positions $t'$ [S1].

**Numbered Steps:**
1.  **Sample Free-Flight Distance:** Sample a distance $t$ from the distribution proportional to $\sigma_t(t) T(t)$ using delta tracking [S1].
    *   If $t < t_s$: The interaction occurs within the medium.
    *   If $t \geq t_s$: The ray hits the surface or exits the medium.
2.  **Compute Contribution for Medium Interaction ($t < t_s$):** If the sampled distance is within the medium, compute the in-scattering gradient contribution [S1].
3.  **Compute Contribution for Surface Interaction ($t \geq t_s$):** If the ray hits the surface, compute the surface contribution [S1].
4.  **Accumulate Transmittance Gradients:** For all null-scattering interactions at locations $t'$ along the ray segment $(0, \min(t, t_s))$, accumulate the transmittance gradient contribution [S1].

**Formulas:**
If $t < t_s$, the contribution is [S1]:
$$ \langle \partial L_1^{\text{DT}} \rangle = \frac{\partial_\theta [\sigma_t(t) \alpha(t)]}{\sigma_t(t)} L_s(t) + \alpha(t) \partial_\theta L_s(t) $$
If $t \geq t_s$, the contribution is [S1]:
$$ \langle \partial L_2^{\text{DT}} \rangle = \partial_\theta L_e(t_s) + \partial_\theta L_s(t_s) $$
The transmittance gradient contribution is [S1]:
$$ \langle \partial L_3^{\text{DT}} \rangle = \begin{cases} -\partial_\theta \sigma_t(t') \alpha(t) L_s(t) & \text{if } t < t_s, \\ -\partial_\theta \sigma_t(t') [L_e(t_s) + L_s(t_s)] & \text{otherwise.} \end{cases} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_dt_gradient(t, null_collisions, t_s, theta, sigma_t_fn, alpha_fn, L_s_fn, L_e_fn, d_param):
    """
    Computes gradient estimates for a single sampled free-flight distance.
    t: sampled free-flight distance
    null_collisions: list of positions t' where null-scattering occurred
    t_s: surface intersection or medium boundary distance
    theta: scene parameters
    sigma_t_fn(t): returns total extinction coefficient at t
    alpha_fn(t): returns scattering albedo at t
    L_s_fn(t): returns in-scattered radiance at t
    L_e_fn(t): returns emitted radiance at t
    d_param(p, theta): returns partial derivative of parameter p w.r.t. theta
    """
    grad = 0.0
    for tp in null_collisions:
        if t < t_s:
            grad += -d_param(sigma_t_fn(tp), theta) * alpha_fn(t) * L_s_fn(t)
        else:
            grad += -d_param(sigma_t_fn(tp), theta) * (L_e_fn(t_s) + L_s_fn(t_s))

    if t < t_s:
        grad += (d_param(sigma_t_fn(t) * alpha_fn(t), theta) / sigma_t_fn(t)) * L_s_fn(t) + alpha_fn(t) * d_param(L_s_fn(t), theta)
    else:
        grad += d_param(L_e_fn(t_s), theta) + d_param(L_s_fn(t_s), theta)
    return grad
```

**Parameters:**
*   $\sigma_t(t)$: Total extinction coefficient [S1].
*   $\alpha(t)$: Scattering albedo, $\sigma_s(t) / \sigma_t(t)$ [S1].
*   $L_s(t)$: In-scattered radiance [S1].
*   $L_e(t)$: Emitted radiance [S1].

**Edge Cases and Pitfalls:**
*   **Bias at Zero Density:** If $\sigma_t(t) = 0$, the estimator $\langle \partial L_1^{\text{DT}} \rangle$ is never sampled because the sampling probability is zero, even though the integrand for in-scattering gradients is non-zero. This causes bias, preventing optimization from escaping states where $\sigma_t=0$ [S1].
*   **High Variance at Low Density:** If $\sigma_t(t)$ is close to zero, the term $1/\sigma_t(t)$ in the estimator leads to large outliers. The variance is unbounded for a homogeneous slab [S1].
*   **Clamping Limitation:** Unlike surface BSDF optimization, clamping $\sigma_t$ to a small nonzero value is not a valid solution here. Lowering $\sigma_t$ increases brightness, which is the wrong direction if the background is darker than the target, leading to incorrect gradient updates [S1].

## Subtopics

### Unbiased Estimators (DRT)
DRT (Delta Tracking with Transmittance sampling) replaces the problematic $\langle \partial L_1^{\text{DT}} \rangle$ estimator with tailored estimators that sample distances proportional to transmittance $T(t')$ to address bias and variance issues [S1]. It samples $t' \sim T(t')$ to estimate in-scattering gradients, while keeping $t \sim \sigma_t T$ for other terms [S1]. The in-scattering gradient estimator is $\langle \partial L_{1a}^{\text{DRT}} \rangle := L_s(t') \partial_\theta [\sigma_t(t') \alpha(t')]$ and the albedo gradient estimator is $\langle \partial L_{1b}^{\text{DRT}} \rangle := \alpha \partial_\theta L_s(t)$ [S1]. A key drawback is that because $t'$ is sampled from a different strategy than the path construction, path replay backpropagation cannot efficiently produce an estimate of incident illumination $L_i$ at $t'$, potentially raising time complexity to $O(n^2)$ [S1].

### Multiple Importance Sampling for Gradient Estimation
This variant combines the standard DT estimator (sampling proportional to $\sigma_t T$) with the DRT estimator (sampling proportional to $T$) to reduce variance in regions of low throughput without increasing complexity [S1]. The PDFs are $p_{\text{DT}}(t) = \sigma_t(t) T(t)$ and $p_{\text{DRT}}(t) = T(t)$. Using the power heuristic (power=2), the MIS weights are $w_{\text{DT}}(t) = \frac{\sigma_t(t)^2}{\sigma_t(t)^2 + 1}$ and $w_{\text{DRT}}(t) = \frac{1}{\sigma_t(t)^2 + 1}$ [S1]. The PDF $p_{\text{DRT}}(t)$ is unnormalized, so variance reduction is not guaranteed by standard MIS bounds, though significant reduction is observed in practice [S1]. Using the novel estimator once per path lowers complexity back to linear time, and MIS corrects the variance increase at negligible cost [S1].

## Operating characteristics
*   **Complexity:** Standard DT estimators are $O(n)$ per path, where $n$ is the number of null collisions. DRT estimators may incur $O(n^2)$ complexity in the worst case due to the need for recursive path starting at sampled points $t'$ [S1].
*   **Variance:** Standard DT estimators suffer from unbounded variance when density is low. DRT estimators address bias and reduce variance in these regions, while MIS combines both strategies to correct variance at negligible cost [S1].
*   **Strengths:** Handles variable density media effectively; integrates naturally with delta tracking; MIS variant maintains linear complexity with low variance [S1].
*   **Limitations:** Standard DT is biased and high-variance in low-density regions; DRT is computationally more expensive due to sampling strategy mismatch; unnormalized DRT PDF means MIS bounds do not strictly apply [S1].

## Background
The method was introduced for inverse volume rendering to address limitations in applying primal sampling techniques directly to the adjoint problem [S1]. It builds upon earlier foundations in differential volume rendering [S1].

## Sources
[S1] Unbiased inverse volume rendering with differential trackers, https://doi.org/10.1145/3528223.3530073, fetched 2026-11-01
