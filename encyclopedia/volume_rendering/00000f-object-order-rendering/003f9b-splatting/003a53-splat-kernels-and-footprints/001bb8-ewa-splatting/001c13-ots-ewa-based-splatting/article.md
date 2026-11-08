# OTS (EWA-Based Splatting)

Also called: EWA-Based Splatting

## Overview
OTS (Opacity-Thin-Side) is a splatting method using unnormalized Gaussians that preserves the 2D amplitude $a'$ (called 'opacity') across projections, unlike EWA Splatting which preserves the weight $w$ [S1]. It differs from standard EWA Splatting by requiring a Jacobian determinant correction to preserve the integral of the extinction coefficient under perspective projection [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ 3D Gaussian primitives, each defined by mean $\boldsymbol{\mu}_i \in \mathbb{R}^3$, covariance $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$, weight $w_i$, and color $c_i$ [S1].
*   **Output:** A 2D image $I(\mathbf{p})$ [S1].

**Data Structures:**
*   **Primitive:** Stores $\boldsymbol{\mu}_i$, $\boldsymbol{\Sigma}_i$, $w_i$, and $c_i$ [S1].
*   **Projected Primitive:** Stores $\boldsymbol{\mu}'_i \in \mathbb{R}^2$, $\boldsymbol{\Sigma}'_i \in \mathbb{R}^{2 \times 2}$, and amplitude $a'_i$ [S1].

**Steps:**
1.  **Projection:** Compute the projected 2D mean $\boldsymbol{\mu}'_i$ and covariance $\boldsymbol{\Sigma}'_i$ in screen space using an affine approximation of the perspective projection [S1].
2.  **Amplitude & Jacobian Correction:** Compute $a'_i$ following Eq. 20, then multiply by the determinant of the Jacobian of the perspective projection to preserve the integral of the extinction coefficient, as $\boldsymbol{\Sigma}'$ varies with screen-space position [S1].
3.  **Rendering:** For each pixel $\mathbf{p}$, accumulate contributions from primitives sorted by depth [S1].

**Formulas:**
The projected 2D Gaussian function $f_i(\mathbf{p})$ is defined as the marginal distribution of the 3D Gaussian along the view axis [S1]:
$$
f_i(\mathbf{p}) = \mathcal{G}_2^n(\mathbf{p}, w_i, \boldsymbol{\mu}'_i, \boldsymbol{\Sigma}'_i) = \int_{-\infty}^{\infty} \mathcal{G}_3^n(\mathcal{R}(t), w_i, \boldsymbol{\mu}_i, \boldsymbol{\Sigma}_i) \, dt
$$
The projected opacity $o_i(\mathbf{p})$ preserves the 2D amplitude $a'$ across projections [S1]:
$$
o_i(\mathbf{p}) = \mathcal{G}_2^u(\mathbf{p}, a'_i, \boldsymbol{\mu}'_i, \boldsymbol{\Sigma}'_i)
$$
The final image function $I(\mathbf{p})$ results from approximations of no overlap, sorted order, uniform color per Gaussian, affine projection, and first-order Taylor expansion of attenuation ($e^x \approx 1-x$) [S1]:
$$
I(\mathbf{p}) = \sum_{i=0}^N c_i(\mathcal{R})g_i(\mathbf{p}) \prod_{j=0}^{i-1} (1 - g_j(\mathbf{p})) + c_b \prod_{i=0}^N (1 - g_i(\mathbf{p}))
$$
where $c_i(\mathcal{R})$ is the view-dependent color evaluated at the mean, $g_i(\mathbf{p}) = \min(\max(f_i(\mathbf{p}), 0.0), 1.0)$ is the clamped opacity used in compositing, and $c_b$ is the background color [S1].

**Parameters:**
*   **Clamp:** $f_i(\mathbf{p})$ must be clamped to $[0, 1]$ because 2D Gaussians can produce values greater than 1 [S1].

**Edge cases and pitfalls:**
*   The clamp on $f_i(\mathbf{p})$ is required for correctness in the OTS model, whereas it was previously only used in the 3DGS codebase for numerical stability [S1].
*   Degenerate covariances (zero determinant) must be skipped to avoid division by zero during Gaussian evaluation [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def evaluate_gaussian_2d(p, mu_prime, Sigma_prime, a_prime):
    # p: (x, y), mu_prime: (mx, my), Sigma_prime: [[s11, s12], [s21, s22]]
    det = Sigma_prime[0][0] * Sigma_prime[1][1] - Sigma_prime[0][1] * Sigma_prime[1][0]
    if det <= 0:
        return 0.0
    inv_det = 1.0 / det
    inv_Sigma = [[Sigma_prime[1][1] * inv_det, -Sigma_prime[0][1] * inv_det],
                 [-Sigma_prime[1][0] * inv_det, Sigma_prime[0][0] * inv_det]]
    diff = (p[0] - mu_prime[0], p[1] - mu_prime[1])
    dist = (diff[0] * (inv_Sigma[0][0] * diff[0] + inv_Sigma[0][1] * diff[1]) +
            diff[1] * (inv_Sigma[1][0] * diff[0] + inv_Sigma[1][1] * diff[1]))
    norm_factor = 1.0 / (2.0 * math.pi * math.sqrt(det))
    f_i = norm_factor * math.exp(-0.5 * dist)
    return min(max(f_i, 0.0), 1.0)

def render_pixel(p, primitives_sorted, background_color):
    # Primitives must be sorted back-to-front by depth [S1]
    accumulated_opacity = 0.0
    accumulated_color = (0.0, 0.0, 0.0)
    for prim in primitives_sorted:
        # Color c_i is evaluated at the mean per source [S1]
        color = prim['color']
        f_i = evaluate_gaussian_2d(p, prim['mu_prime'], prim['Sigma_prime'], prim['a_prime'])
        g_i = f_i
        alpha = g_i
        accumulated_color = (
            accumulated_color[0] + (1.0 - accumulated_opacity) * alpha * color[0],
            accumulated_color[1] + (1.0 - accumulated_opacity) * alpha * color[1],
            accumulated_color[2] + (1.0 - accumulated_opacity) * alpha * color[2]
        )
        accumulated_opacity += (1.0 - accumulated_opacity) * alpha
        if accumulated_opacity > 0.999:
            break
    final_color = (
        accumulated_color[0] + (1.0 - accumulated_opacity) * background_color[0],
        accumulated_color[1] + (1.0 - accumulated_opacity) * background_color[1],
        accumulated_color[2] + (1.0 - accumulated_opacity) * background_color[2]
    )
    return final_color
```

## Subtopics

### [Self-Attenuation Rendering](1154-self-attenuation-rendering.md)
Self-Attenuation Rendering is a volumetric splatting technique that accurately accounts for the exponential extinction of a Gaussian primitive along the viewing ray, rather than ignoring it or using a first-order Taylor approximation. It solves the problem of visual inaccuracies and training instability caused by the standard approximation $1-g(\mathbf{p})$ for attenuation, which fails when the extinction coefficient $g(\mathbf{p})$ exceeds 1. Unlike standard EWA Splatting and 3DGS, which approximate the attenuation term, this method uses the exact exponential form $1-e^{-f(\mathbf{p})}$, remaining valid for any $f(\mathbf{p}) \in [0, \infty)$ without requiring clamping. Implementing this requires switching from back-to-front traversal to front-to-back traversal in the backward pass.

### 3D Gaussian Splatting (3DGS) Approximation
3DGS Approximation is a splatting method using unnormalized Gaussians that preserves 2D amplitude $a'$ (called 'opacity') across projections, unlike EWA which preserves weight $w$ [S1]. It models projected opacity as $o_i(\mathbf{p}) = \mathcal{G}_2^u(\mathbf{p}, a'_i, \boldsymbol{\mu}'_i, \boldsymbol{\Sigma}'_i)$ [S1]. The final image function results from approximations of no overlap, sorted order, uniform color per Gaussian, affine projection, and first-order Taylor expansion of attenuation ($e^x \approx 1-x$) [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Strengths:** Preserves the 2D amplitude $a'$ across projections and applies a Jacobian determinant correction to maintain the integral of the extinction coefficient under perspective projection [S1].
*   **Limitations:** Requires explicit clamping of $f_i(\mathbf{p})$ for mathematical correctness [S1].
*   **When to use:** Not given in the sources [S1].
*   **When not:** Not given in the sources [S1].

## Background
OTS (Opacity-Thin-Side) is a variant of EWA Splatting that builds on [EWA Splatting](1069-ewa-splatting.md) [S1]. The method was introduced in "Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?" [S1].

## Sources
[S1] Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?, https://doi.org/10.1111/cgf.70032, fetched 2026-10-26
