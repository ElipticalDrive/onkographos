# Weighted Blending (Fuzzy Metaballs)
Also called: Blending intersections, Fuzzy Metaballs rendering, Hidden-surface solution for Fuzzy Metaballs, Sort-free Gaussian rendering, Weighted Blending of Intersections

## Overview
Weighted Blending is a sort-free rendering technique for 3D Gaussians that resolves the hidden-surface problem by computing a weighted average of intersection depths, avoiding the computational cost of sorting Gaussians by depth [S1, unofficial][S2]. Unlike NeRF-style methods that sort intersections and use closer Gaussians to attenuate further ones, this method blends all intersections simultaneously, treating the Gaussian mixture as an implicit surface where the final depth is a density-weighted mean of all ray-Gaussian intersections [S1, unofficial][S2].

## How it works
**Inputs:** Ray direction $\vec{v}$ (normalized), list of $N$ 3D Gaussians. Each Gaussian $i$ is defined by mean $\mu_i \in \mathbb{R}^3$, inverse covariance $\Sigma_i^{-1} \in \mathbb{R}^{3 \times 3}$, and weight $\lambda_i \ge 0$ with $\sum_i \lambda_i = 1$ [S1, unofficial].
**Outputs:** Blended intersection depth $t_f$, opacity $\alpha$, and optionally normal $\vec{n}_f$ or color $\vec{c}_f$ [S1, unofficial].

**Data Structures:**
*   `Gaussian`: fields `mu` (vec3), `Sigma_inv` (mat3x3), `lambda` (float).
*   `Ray`: fields `origin` (vec3), `direction` (vec3).

**Steps:**
1.  For each Gaussian $i$ in the scene:
    a.  Compute the intersection distance $t_i$ along the ray $\vec{v}$ using the linear approach (point of maximum likelihood) [S1, unofficial]:
        $$t_i = \frac{\mu_i^T \Sigma_i^{-1} \vec{v}}{\vec{v}^T \Sigma_i^{-1} \vec{v}}$$
    b.  Compute the unnormalized log distance $d_i$:
        $$d_i = -\frac{1}{2} \left[ (t_i \vec{v} - \mu_i)^T \Sigma_i^{-1} (t_i \vec{v} - \mu_i) \right] + \log(\lambda_i)$$
    c.  Compute the unnormalized Gaussian density $\delta_i$:
        $$\delta_i = \exp(d_i)$$
    d.  Compute the weight $w_i$ using the hyperparameters $\beta_1, \beta_2, \beta_3$ and shape scale $\eta$ [S1, unofficial][S2]:
        $$w_i = \exp \left( \beta_1 d_i \sigma \left( \frac{\beta_3}{\eta} t_i \right) - \frac{\beta_2}{\eta} t_i \right)$$
        where $\sigma(x) = \frac{1}{1 + e^{-x}}$ is the sigmoid function [S1, unofficial].
2.  Compute the final blended depth $t_f$:
    $$t_f = \frac{\sum_i w_i t_i}{\sum_i w_i}$$
3.  Compute the opacity $\alpha$ [S1, unofficial]:
    $$\alpha = \sigma \left( \beta_4 \sum_i \delta_i + \beta_5 \right)$$

**Parameters:**
*   $\beta_1, \beta_2, \beta_3, \beta_4, \beta_5$: Hyperparameters controlling weight and transparency functions [S1, unofficial]. $\beta_1$ and $\beta_2$ balance high-quality hits versus hits closer to the camera; $\beta_3$ controls the sigmoid decay for behind-camera intersections [S2].
*   $\eta$: Rough scale of the object, used to normalize $t_i$ in the weight calculation to ensure rendering invariance to object scale [S1, unofficial][S2].
*   Typical values: The sources state that all five hyperparameters ($\beta_{1,2,3,4,5}$) were held constant throughout experiments, but do not provide specific numerical values [S2].

**Edge Cases and Pitfalls:**
*   **Behind Camera:** Intersections with $t_i < 0$ are behind the camera. The sigmoid term $\sigma(\frac{\beta_3}{\eta} t_i)$ down-weights these intersections [S2]. If $\beta_3=0$, this term becomes 1, ignoring the behind-camera penalty.
*   **Precision:** The calculation involves exponentials of potentially large negative numbers ($d_i$). Underflow to zero is possible for distant or poorly aligned Gaussians, effectively removing them from the blend [S1, unofficial].
*   **Normalization:** The original formulation assumes $\sum \lambda_i = 1$. Deviations from this (e.g., in simplified models) may cause minor differences in pose estimation but are negligible for shape estimation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sigmoid(x):
    return 1.0 / (1.0 + math.exp(-x))

def weighted_blend_gaussians(ray_origin, ray_dir, gaussians, beta1, beta2, beta3, beta4, beta5, eta):
    """
    ray_origin: vec3
    ray_dir: vec3 (normalized)
    gaussians: list of dicts with keys 'mu' (vec3), 'Sigma_inv' (mat3x3), 'lambda' (float)
    beta1, beta2, beta3, beta4, beta5: float hyperparameters
    eta: float shape scale
    """
    sum_w = 0.0
    sum_wt = 0.0
    sum_delta = 0.0
    
    for g in gaussians:
        mu = g['mu']
        Sigma_inv = g['Sigma_inv']
        lam = g['lambda']
        
        # Step 1a: Compute intersection distance t_i
        # t_i = (mu^T Sigma_inv v) / (v^T Sigma_inv v)
        # Note: mu^T Sigma_inv v is a scalar dot product of (Sigma_inv^T mu) and v
        # Since Sigma_inv is symmetric, Sigma_inv^T = Sigma_inv
        numerator = dot(Sigma_inv_mul_vec(Sigma_inv, mu), ray_dir)
        denominator = dot(Sigma_inv_mul_vec(Sigma_inv, ray_dir), ray_dir)
        
        if denominator == 0:
            continue
            
        t_i = numerator / denominator
        
        # Step 1b: Compute unnormalized log distance d_i
        # d_i = -0.5 * (t_i*v - mu)^T Sigma_inv (t_i*v - mu) + log(lambda)
        diff = scale_vec(ray_dir, t_i)
        diff = sub_vec(diff, mu)
        quad_form = dot(Sigma_inv_mul_vec(Sigma_inv, diff), diff)
        d_i = -0.5 * quad_form + math.log(lam)
        
        # Step 1c: Compute density delta_i
        delta_i = math.exp(d_i)
        sum_delta += delta_i
        
        # Step 1d: Compute weight w_i
        # w_i = exp( beta1 * d_i * sigmoid(beta3/eta * t_i) - beta2/eta * t_i )
        sigmoid_term = sigmoid((beta3 / eta) * t_i)
        exponent = beta1 * d_i * sigmoid_term - (beta2 / eta) * t_i
        w_i = math.exp(exponent)
        
        sum_w += w_i
        sum_wt += w_i * t_i
        
    # Step 2: Compute final depth
    if sum_w == 0:
        return 0.0, 0.0 # Handle no intersections
        
    t_f = sum_wt / sum_w
    
    # Step 3: Compute opacity
    alpha = sigmoid(beta4 * sum_delta + beta5)
    
    return t_f, alpha

def Sigma_inv_mul_vec(Sigma_inv, v):
    """Matrix-vector multiplication for 3x3 matrix and 3-vector"""
    return [
        Sigma_inv[0][0]*v[0] + Sigma_inv[0][1]*v[1] + Sigma_inv[0][2]*v[2],
        Sigma_inv[1][0]*v[0] + Sigma_inv[1][1]*v[1] + Sigma_inv[1][2]*v[2],
        Sigma_inv[2][0]*v[0] + Sigma_inv[2][1]*v[1] + Sigma_inv[2][2]*v[2]
    ]

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def scale_vec(v, s):
    return [v[0]*s, v[1]*s, v[2]*s]

def sub_vec(v1, v2):
    return [v1[0]-v2[0], v1[1]-v2[1], v1[2]-v2[2]]
```

## Subtopics

### Two Parameter Model
A simplified rendering model for shape reconstruction that reduces the original 5 hyperparameters to 2 ($\beta_1, \beta_2$) [S1, unofficial]. It removes the sigmoid term for behind-camera contributions by setting $\beta_3 = 0$, as intersections behind the camera are not used in practice [S1, unofficial]. It replaces the sigmoid alpha function with a decaying exponential, dropping $\beta_4$ and $\beta_5$, and relying on unnormalized $\lambda_i$ for opacity control [S1, unofficial]. The weight function becomes $w_i = \exp \left( \beta_1 d_i - \frac{\beta_2}{\eta} t_i \right)$ and alpha becomes $\alpha = 1 - \exp \left( - \sum_i \delta_i \right)$ [S1, unofficial]. Losing the global normalization condition ($\sum \lambda_i = 1$) causes minor differences in pose estimation but is negligible for shape estimation [S1, unofficial].

### Zero Parameter Model
A hyperparameter-free rendering variant that requires sorting intersections and uses alpha compositing (transmission) to compute weights [S1, unofficial]. It sorts all ray-Gaussian intersections by depth $t_i$ and computes accumulated transmission $T_i$ for each intersection based on earlier (closer) intersections [S1, unofficial]. The weight is $w_i = T_i (1 - \exp(-\delta_i))$ where $T_i = \exp \left( -\sum_j \delta_j 1[t_j < t_i] \right)$ [S1, unofficial]. This approach is slower than the sort-free models but performs nearly identically in quality and requires no hyperparameters [S1, unofficial].

### Fuzzy Metaballs
An algebraic, implicit surface representation where the surface is defined by the zero level set of a function $F(x,y,z)$ [S2]. It uses multidimensional Gaussians (ellipsoids) as components instead of isotropic Gaussians, relaxing the hard threshold $T$ to allow ray-tracing formulation where all rays intersect all Gaussians, ensuring smooth gradients [S2].

## Operating characteristics
*   **Time Complexity:** Requires $O(N)$ evaluations of Gaussians for each ray, where $N$ is the number of Gaussians [S2].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On a Quadro P2000 (CUDA), rendering 160x120 images takes $3.0 \pm 0.2$ ms forward and $9.6 \pm 0.5$ ms backward. On an i5-7287U (CPU), rendering 80x60 images takes $3.0 \pm 0.15$ ms forward and $13.2 \pm 1.4$ ms backward [S2]. This is faster than Point Cloud (12.1 ms forward CUDA), Pulsar (7.8 ms forward CUDA), and SoftRas Mesh (17.0 ms forward CUDA) [S2].
*   **Strengths:** Sort-free, enabling efficient parallelization on GPUs without complex sorting networks. Robust and fast [S2].
*   **Limitations:** The zero-parameter variant (sorting-based) is slower [S1, unofficial]. The weighted blending variant requires tuning of hyperparameters ($\beta$ values) and the scale factor $\eta$ [S1, unofficial][S2].
*   **When to use:** When rendering 3D Gaussian mixtures as implicit surfaces is desired, and sorting overhead is prohibitive. Suitable for shape reconstruction from video [S1, unofficial].
*   **When not to use:** When strict physical transparency compositing is required (use Zero Parameter Model or standard OIT) or when hyperparameter tuning is undesirable.

## Background
The method originates from "Fuzzy Metaballs" by Leonid Keselman and Martial Hebert, published in 2022 [S2]. It was introduced as an algebraic, implicit surface representation using multidimensional Gaussians [S2]. The weighted blending technique was later adapted and analyzed in the context of differentiable rendering with 3D Gaussians, specifically in "Flexible Techniques for Differentiable Rendering with 3D Gaussians" (2023) [S1, unofficial]. The technique is related to Order Independent Transparency (OIT) but extended to 3D opaque surfaces [S2].

## Sources
[S1] Flexible Techniques for Differentiable Rendering with 3D Gaussians, https://doi.org/10.48550/arxiv.2308.14737, fetched 2026-10-26 (unofficial)
[S2] Approximate Differentiable Rendering with Algebraic Surfaces, https://doi.org/10.1007/978-3-031-19824-3_35, fetched 2026-10-26
