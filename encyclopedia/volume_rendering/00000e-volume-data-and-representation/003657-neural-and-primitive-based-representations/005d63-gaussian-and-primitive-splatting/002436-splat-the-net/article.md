# Splat the Net

Also called: Neural Primitives, Splat the Net: Radiance Fields with Splattable Neural Primitives

## Overview
Splat the Net is a neural primitive-based volume rendering method that replaces Gaussian primitives with splattable neural density fields to represent 3D scenes [S1, unofficial]. It solves the trade-off between representation expressivity and rendering speed by using MLP-based primitives that require roughly 10× fewer primitives than 3DGS for the same scene representation while maintaining >100 FPS rendering speed [S1, unofficial]. Unlike its sibling [Neural Primitive Splatting](3300-neural-primitive-splatting.md) which uses fixed Gaussian kernels, Splat the Net uses an 8-neuron MLP to define the density profile of each primitive, offering greater expressivity under identical memory budgets [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D spatial position $\mathbf{x}$, viewing direction $\mathbf{d}$, and optionally time $\xi_t$ or light direction $\mathbf{d}_l$ [S1, unofficial].
*   **Output:** Density $\sigma$ and color $\mathbf{c}$ for a ray passing through the scene [G].

**Data Structures:**
Each primitive $i$ stores 99 parameters total [S1, unofficial]:
1.  **Geometry (10 parameters):** mean position $\boldsymbol{\mu}_i \in \mathbb{R}^3$, scale $\mathbf{s}_i \in \mathbb{R}^3$, rotation quaternion $\mathbf{q}_i \in \mathbb{R}^4$ [S1, unofficial].
2.  **Neural Density Field (41 parameters):** an 8-neuron MLP defining the radial density profile with weights $W_1, W_2$ and biases $b_1, b_2$ [S1, unofficial]. For dynamic scenes, an additional temporal weight $W_t \in \mathbb{R}^{N_\sigma}$ is included [S1, unofficial].
3.  **Appearance (48 parameters):** Spherical Harmonic (SH) coefficients for color [S1, unofficial].

**Numbered Steps:**

1.  **Initialization:** Uniformly sample $N$ primitives within the scene's bounding box. For dynamic scenes, $N=100,000$ [S1, unofficial].
2.  **Density Evaluation:** For a query point $\mathbf{x}$ relative to primitive $i$, compute the local coordinate $\mathbf{x}' = \mathbf{R}_i^T (\mathbf{x} - \boldsymbol{\mu}_i)$, where $\mathbf{R}_i$ is the rotation matrix derived from $\mathbf{q}_i$ [G]. Normalize by scale $\mathbf{s}_i$ to get the normalized distance [G]. Pass the normalized distance through the MLP to get density $\sigma_i(\mathbf{x})$ [S1, unofficial].
    *   Static density formula: $\sigma_i(\mathbf{x}) = \text{MLP}(\|\mathbf{x}'\|)$ [G].
    *   Dynamic density formula (Eulerian approach): $f_*(\mathbf{x}, \xi_t) = W_2 \cos(\omega_0 \cdot W_1(\mathbf{x}) + \xi_t \cdot W_t + \mathbf{b}_1) + \mathbf{b}_2$, where $\xi_t \in [0,1]$ is the temporal variable and $W_t$ is a learnable temporal weight [S1, unofficial].
3.  **Color Evaluation:**
    *   **Static/Novel View Synthesis:** Color is determined by the zero-order SH coefficient of the primitive, modulated by the view direction $\mathbf{d}$ [S1, unofficial].
    *   **Dynamic:** The zero-order SH coefficient $S(\xi_t)$ is modeled as a function of time: $S(\xi_t) = S_0 + P_n(\xi_t) + F_l(\xi_t)$, where $P_n(\xi_t) = \sum_{i=1}^n a_i \xi_t^i$ is a polynomial of degree $n$, and $F_l(\xi_t)$ is a Fourier series of order $l$. In experiments, $n=4$ and $l=4$ [S1, unofficial].
    *   **Relighting:** Color is formulated as $\mathbf{c} = \mathbf{c}_{dc} + \mathbf{c}(\mathbf{x}, \mathbf{d}, \mathbf{d}_l)$, where $\mathbf{c}_{dc}$ is a constant color and $\mathbf{c}(\cdot)$ is a neural color function dependent on position, view direction, and light direction $\mathbf{d}_l$ [S1, unofficial].
4.  **Rendering:** Perform volumetric rendering (e.g., alpha compositing) by sampling points along rays, evaluating $\sigma_i$ and $\mathbf{c}_i$ for all primitives, and accumulating opacity and color [G].
5.  **Population Control (Densification and Pruning):**
    *   **Densification:** Split or clone primitives based on gradient magnitude [G].
    *   **Pruning:** Remove primitives with low opacity or large scale [G].
    *   **Thresholds:** Growing scale threshold $10^{-2}$, pruning scale threshold $0.5$ [S1, unofficial]. MLP-gradient-based densification threshold $10^{-4}$ (Mip-NeRF360/Tanks&Temples/Deep Blending), $10^{-5}$ (BlenderNeRF) [S1, unofficial]. MLP-gradient-based pruning threshold $2 \times 10^{-6}$ (Mip-NeRF360/Tanks&Temples/Deep Blending), $10^{-6}$ (BlenderNeRF) [S1, unofficial].
    *   **Frequency:** Mip-NeRF360/Tanks&Temples/Deep Blending every 500 iterations between 1k and 15k [S1, unofficial]. BlenderNeRF every 200 iterations between 1k and 20k [S1, unofficial]. Dynamic scenes every 500 iterations between 1k and 30k [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def evaluate_density(x_local, params, is_dynamic=False, xi_t=0.0):
    """Evaluates density for a neural primitive at normalized local distance x_local."""
    z = np.dot(params['W1'], x_local) + params['b1']
    if is_dynamic and 'Wt' in params:
        z += np.dot(params['Wt'], xi_t)
    density = params['W2'] * np.cos(params['omega0'] * z) + params['b2']
    return np.maximum(density, 0.0)

def evaluate_color(sh_coeffs, view_dir, is_dynamic=False, xi_t=0.0, n=4, l=4):
    """Evaluates color from SH coefficients, modulating zero-order for dynamic scenes."""
    color = np.zeros(3)
    # Standard SH evaluation [G]
    for i in range(len(sh_coeffs)):
        color += sh_coeffs[i] * sh_basis(i, view_dir)
    if is_dynamic:
        # Modulate zero-order SH coefficient S0 with time
        poly = sum(sh_coeffs[i] * (xi_t ** i) for i in range(n + 1))
        fourier = sum(sh_coeffs[n + 1 + 2*k] * np.cos(k * np.pi * xi_t) + 
                      sh_coeffs[n + 2 + 2*k] * np.sin(k * np.pi * xi_t) for k in range(l))
        color += poly + fourier
    return color

def sh_basis(order, dir):
    """Standard spherical harmonic basis function [G]."""
    x, y, z = dir
    if order == 0: return 0.282095
    elif order == 1: return 0.488603 * z
    elif order == 2: return 1.092548 * x * z
    elif order == 3: return 0.946175 * y * z
    elif order == 4: return 0.546274 * (3 * z**2 - 1)
    return 0.0

def render_ray(origin, direction, primitives, is_dynamic=False, xi_t=0.0):
    """Renders a single ray by alpha compositing neural primitives."""
    # Intersect ray with scene bounding box to find t_min, t_max [G]
    # Sample points along ray within bounds [G]
    t_vals = np.linspace(t_min, t_max, num_samples)
    alpha_accum = 0.0
    color_accum = np.zeros(3)
    
    for t in t_vals:
        point = origin + t * direction
        sigma_total = 0.0
        color_weighted = np.zeros(3)
        
        for prim in primitives:
            # Transform point to primitive local space
            local_x = prim['R'].T @ (point - prim['mu'])
            norm_dist = np.linalg.norm(local_x) / np.linalg.norm(prim['s'])
            
            sigma = evaluate_density(norm_dist, prim['mlp_params'], is_dynamic, xi_t)
            c = evaluate_color(prim['sh_coeffs'], direction, is_dynamic, xi_t)
            
            alpha = 1.0 - np.exp(-sigma * ray_step_size)
            color_weighted += alpha * c * (1.0 - alpha_accum)
            alpha_accum += alpha * (1.0 - alpha_accum)
            
        color_accum += color_weighted
    return color_accum
```

**Parameters:**
*   MLP learning rate: $10^{-3}$ [S1, unofficial].
*   Primitive means learning rate: $1.6 \times 10^{-4}$ [S1, unofficial].
*   Primitive scales learning rate: $5 \times 10^{-3}$ [S1, unofficial].
*   Primitive quaternions learning rate: $10^{-3}$ [S1, unofficial].
*   SH coefficients learning rate: $2.5 \times 10^{-3}$ [S1, unofficial].
*   Polynomial degree $n$ and Fourier series order $l$: 4 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The method requires higher FLOPs per integration (769 FLOPs) compared to 3DGS (43 FLOPs) [S1, unofficial]. Implementation must optimize the MLP evaluation to maintain real-time performance [S1, unofficial].
*   **Boundaries:** Densification and pruning thresholds vary by dataset (BlenderNeRF vs. Mip-NeRF360/etc.). Using incorrect thresholds can lead to over-sampling or under-representation [S1, unofficial].
*   **Degenerate Input:** The temporal variable $\xi_t$ must be normalized to $[0,1]$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Higher computational cost per primitive than Gaussian primitives (769 FLOPs vs 43 FLOPs) [S1, unofficial]. However, due to requiring 10× fewer primitives, overall inference speed remains >100 FPS, comparable to 3DGS [S1, unofficial].
*   **Memory Complexity:** Lower memory usage than 3DGS for equivalent quality. Requires half the number of primitives for the same memory budget [S1, unofficial].
*   **Strengths:** Higher expressivity than Gaussian primitives under the same memory budget; real-time rendering speed; supports dynamic scenes and relighting [S1, unofficial].
*   **Limitations:** Higher per-primitive computational cost; requires careful tuning of densification/pruning thresholds [S1, unofficial].
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
Splat the Net was introduced in the paper "Splat the Net: Radiance Fields with Splattable Neural Primitives" [S1, unofficial]. It builds on the principles of neural and primitive-based representations [G]. The method was developed to address the limitations of Gaussian primitives in 3DGS, specifically their lower expressivity per parameter [S1, unofficial]. It is used for novel view synthesis, dynamic scene reconstruction, and relighting [S1, unofficial].

## Sources
[S1] Splat the Net: Radiance Fields with Splattable Neural Primitives, https://doi.org/10.48550/arxiv.2510.08491, fetched 2026-10-26 (unofficial)
