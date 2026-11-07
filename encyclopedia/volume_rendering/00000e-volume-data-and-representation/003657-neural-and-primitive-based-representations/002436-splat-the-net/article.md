# Splat the Net

Also called: Neural Primitives, Splat the Net: Radiance Fields with Splattable Neural Primitives

## Overview
Splat the Net is a neural primitive-based volume rendering method that replaces the Gaussian kernels of 3D Gaussian Splatting (3DGS) with splattable neural primitives [S1, unofficial]. It represents 3D scenes using a set of parameterized primitives, each containing a small Multi-Layer Perceptron (MLP) for density and Spherical Harmonics (SH) for color, allowing for higher expressivity per primitive [S1, unofficial]. The method achieves comparable real-time rendering speeds (>100 FPS) to 3DGS while requiring approximately 10 times fewer primitives for the same scene representation [S1, unofficial]. It builds on [Neural and primitive-based representations](10-neural-and-primitive-based-representations.md) by introducing neural density fields into the splatting framework [G].

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of $N$ primitives, each defined by geometric parameters and neural weights [S1, unofficial].
*   Camera parameters (position, orientation, intrinsics) [G].
*   For dynamic scenes: a temporal variable $\xi_t \in [0, 1]$ [S1, unofficial].
*   For relighting: a light direction $\mathbf{d}_l$ [S1, unofficial].

**Outputs:**
*   Rendered RGB images for given camera viewpoints [G].

### Data Structures
Each primitive $i$ stores the following parameters, totaling 99 parameters per primitive (compared to 59 for 3DGS) [S1, unofficial]:

1.  **Geometry (10 parameters):**
    *   Mean position $\mathbf{\mu} \in \mathbb{R}^3$.
    *   Scale $\mathbf{s} \in \mathbb{R}^3$.
    *   Rotation represented as a unit quaternion $\mathbf{q} \in \mathbb{R}^4$.
2.  **Neural Density MLP (41 parameters):**
    *   An 8-neuron MLP with specific weights and biases [S1, unofficial].
3.  **Color Representation (48 parameters):**
    *   Zero-order Spherical Harmonics (SH) coefficients [S1, unofficial]. For dynamic scenes, these are time-dependent [S1, unofficial].

### Formulas

**1. Neural Density Field**
The density $\sigma$ at a 3D position $\mathbf{x}$ is computed by a neural primitive. For static scenes, the density is derived from an MLP. For dynamic scenes, an Eulerian approach incorporates time $\xi_t$ [S1, unofficial]:

$$
f_*(\mathbf{x}, \xi_t) = W_2 \cos(\omega_0 \cdot W_1(\mathbf{x}) + \xi_t \cdot W_t + \mathbf{b}_1) + \mathbf{b}_2
$$

Where:
*   $W_1, W_2, \mathbf{b}_1, \mathbf{b}_2$ are learnable weights and biases of the MLP [S1, unofficial].
*   $\omega_0$ is a frequency parameter [G].
*   $W_t \in \mathbb{R}^{N_\sigma}$ is a learnable temporal weight vector [S1, unofficial].
*   $\xi_t \in [0, 1]$ is the normalized timestamp [S1, unofficial].

**2. Dynamic Color (Time-Dependent SH)**
The zero-order SH coefficient $S$ for a primitive is modeled as a function of time $\xi_t$, combining a polynomial and a Fourier series [S1, unofficial]:

$$
S(\xi_t) = S_0 + P_n(\xi_t) + F_l(\xi_t)
$$

Where:
*   $S_0$ is the base zero-order SH coefficient [S1, unofficial].
*   $P_n(\xi_t) = \sum_{i=1}^n a_i \xi_t^i$ is a polynomial of degree $n$ [S1, unofficial].
*   $F_l(\xi_t) = \sum_{k=1}^l (b_k \cos(k \xi_t) + c_k \sin(k \xi_t))$ is a Fourier series of order $l$ [G].
*   In experiments, $n=4$ and $l=4$ [S1, unofficial].

**3. Relighting Color**
For relighting, the primitive color $\mathbf{c}$ per view ray is formulated as a combination of a constant color $\mathbf{c}_{dc}$ and a neural color function dependent on 3D position $\mathbf{x}$, view direction $\mathbf{d}$, and light direction $\mathbf{d}_l$ [S1, unofficial]:

$$
\mathbf{c} = \mathbf{c}_{dc} + \mathbf{c}(\mathbf{x}, \mathbf{d}, \mathbf{d}_l)
$$

### Parameters and Hyperparameters
*   **Optimizer:** Adam [S1, unofficial].
*   **MLP Learning Rate:** $10^{-3}$ [S1, unofficial].
*   **Primitive Means Learning Rate:** $1.6 \times 10^{-4}$ [S1, unofficial].
*   **Primitive Scales Learning Rate:** $5 \times 10^{-3}$ [S1, unofficial].
*   **Primitive Quaternions Learning Rate:** $10^{-3}$ [S1, unofficial].
*   **SH Coefficients Learning Rate:** $2.5 \times 10^{-3}$ [S1, unofficial].
*   **Population Control Growing Scale Threshold:** $10^{-2}$ [S1, unofficial].
*   **Population Control Pruning Scale Threshold:** $0.5$ [S1, unofficial].

**Densification and Pruning Thresholds:**
*   **Mip-NeRF360, Tanks & Temples, Deep Blending:**
    *   Densification Threshold: $10^{-4}$ [S1, unofficial].
    *   Pruning Threshold: $2 \times 10^{-6}$ [S1, unofficial].
    *   Interval: Every 500 iterations between 1k and 15k [S1, unofficial].
*   **BlenderNeRF:**
    *   Densification Threshold: $10^{-5}$ [S1, unofficial].
    *   Pruning Threshold: $10^{-6}$ [S1, unofficial].
    *   Interval: Every 200 iterations between 1k and 20k [S1, unofficial].

### Numbered Steps

1.  **Initialization:**
    *   Uniformly sample $N$ primitives within the scene's bounding box. For dynamic scenes, $N=100,000$ [S1, unofficial].
    *   Initialize MLP weights, geometric parameters ($\mathbf{\mu}, \mathbf{s}, \mathbf{q}$), and SH coefficients.

2.  **Rendering:**
    *   For each pixel, cast a ray.
    *   Sort primitives by depth along the ray.
    *   For each primitive, compute the density $\sigma$ using the neural density field formula.
    *   Compute the color $\mathbf{c}$ using the SH coefficients (adjusted for time $\xi_t$ if dynamic) or the relighting formula.
    *   Perform volume rendering integration (alpha compositing) to obtain the final pixel color.

3.  **Training:**
    *   Render images from training views.
    *   Compute loss against ground truth images.
    *   Backpropagate to update all parameters (MLP weights, geometry, SH coefficients).

4.  **Population Control (Densification and Pruning):**
    *   **Check Interval:** Determine if the current iteration matches the densification interval for the specific dataset (e.g., every 500 iterations for Mip-NeRF360) [S1, unofficial].
    *   **Gradient Analysis:** Compute the gradient of the loss with respect to the MLP weights for each primitive.
    *   **Densification:** If a primitive's gradient magnitude exceeds the densification threshold (e.g., $10^{-4}$) and its scale is below the growing threshold ($10^{-2}$), split the primitive or add new primitives in its vicinity [S1, unofficial].
    *   **Pruning:** If a primitive's scale exceeds the pruning threshold ($0.5$), remove it [S1, unofficial].
    *   **Dynamic Scene Specifics:** Densification starts at iteration 1,000 and continues until iteration 30,000 for dynamic scenes [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def evaluate_density(x, t, params):
    """Computes density using the neural density field.
    x: (3,) spatial coordinate
    t: scalar temporal variable xi_t, None for static
    params: dict with W1, W2, b1, b2, omega0, Wt
    """
    z = params['W1'] @ x
    if t is not None:
        z = z + params['Wt'] * t
    sigma = params['W2'] * np.cos(params['omega0'] * z + params['b1']) + params['b2']
    return np.maximum(sigma, 0.0)  # Density must be non-negative

def evaluate_dynamic_sh(S0, t, poly_coeffs, fourier_b, fourier_c, n=4, l=4):
    """Computes time-dependent SH coefficient.
    S0: scalar base coefficient
    t: scalar temporal variable
    poly_coeffs: (n,) array of a_i
    fourier_b: (l,) array of b_k
    fourier_c: (l,) array of c_k
    """
    poly_term = sum(poly_coeffs[i] * (t ** (i + 1)) for i in range(n))
    fourier_term = sum(fourier_b[k] * np.cos((k + 1) * t) + fourier_c[k] * np.sin((k + 1) * t) for k in range(l))
    return S0 + poly_term + fourier_term

def render_ray(origin, direction, primitives, t=None):
    """Renders a single ray using splatting."""
    # Sort primitives by depth along the ray
    distances = np.array([np.dot(p['center'] - origin, direction) for p in primitives])
    sorted_indices = np.argsort(distances)
    
    alpha_accum = 0.0
    color_accum = np.zeros(3)
    
    for idx in sorted_indices:
        p = primitives[idx]
        # Project primitive to ray
        center_to_ray = p['center'] - origin
        proj_dist = np.dot(center_to_ray, direction)
        closest_point = origin + proj_dist * direction
        dist_to_center = np.linalg.norm(closest_point - p['center'])
        
        # Evaluate density at closest point
        sigma = evaluate_density(closest_point, t, p['mlp_params'])
        
        # Compute opacity from sigma and primitive scale
        alpha = 1.0 - np.exp(-sigma * p['scale'])
        
        if alpha < 1e-5:
            continue
            
        # Compute color
        if t is not None:
            color = evaluate_dynamic_sh(
                p['sh_params']['S0'], t, p['poly_coeffs'], 
                p['fourier_b'], p['fourier_c']
            )
        else:
            color = p['sh_params']['S0']
            
        weight = alpha * (1.0 - alpha_accum)
        color_accum += weight * color
        alpha_accum += weight
        if alpha_accum > 0.99:
            break
            
    return color_accum

def train_step(primitives, batch_images, batch_targets, optimizer, iteration, dataset_cfg):
    """One training step including population control."""
    # Forward render
    rendered_batch = [render_ray(img['origin'], img['direction'], primitives, img.get('t')) for img in batch_images]
    
    # Compute loss (MSE)
    loss = np.mean((np.array(rendered_batch) - np.array(batch_targets)) ** 2)
    
    # Backward pass (simulated gradient update)
    optimizer.step(loss)
    
    # Population control
    interval = dataset_cfg['interval']
    start_iter, end_iter = dataset_cfg['start'], dataset_cfg['end']
    densify_thresh = dataset_cfg['densify_thresh']
    prune_thresh = dataset_cfg['prune_thresh']
    grow_thresh = dataset_cfg['grow_thresh']
    
    if start_iter <= iteration <= end_iter and iteration % interval == 0:
        # Compute gradient norms per primitive (simulated)
        grad_norms = [np.linalg.norm(p.get('grad', np.zeros(3))) for p in primitives]
        
        new_primitives = []
        for i, p in enumerate(primitives):
            if grad_norms[i] > densify_thresh and p['scale'] < grow_thresh:
                # Split: create two children with slightly offset centers and reduced scale
                offset = np.random.randn(3) * p['scale'] * 0.5
                child1 = dict(p, center=p['center'] + offset, scale=p['scale'] * 0.9)
                child2 = dict(p, center=p['center'] - offset, scale=p['scale'] * 0.9)
                new_primitives.extend([child1, child2])
            elif p['scale'] > prune_thresh:
                continue  # Prune
            else:
                new_primitives.append(p)
        primitives = new_primitives
        
    return loss, primitives
```

## Operating characteristics

*   **Computational Cost:** Neural primitives have higher per-primitive computational cost than Gaussian primitives. 3DGS requires ~43 FLOPs per integration, while Splat the Net requires ~769 FLOPs [S1, unofficial].
*   **Rendering Speed:** Despite higher per-primitive cost, the method requires ~10x fewer primitives than 3DGS for the same scene, maintaining >100 FPS rendering speed [S1, unofficial].
*   **Memory Efficiency:** Uses fewer primitives, leading to lower memory usage for equivalent quality. Under limited memory budgets, it outperforms 3DGS [S1, unofficial].
*   **Expressivity:** Neural primitives are more expressive than Gaussian primitives, allowing for better reconstruction of complex structures with fewer primitives [S1, unofficial].
*   **Training Time:** Training involves densification and pruning, which adds overhead. Dynamic scenes require up to 100,000 iterations [S1, unofficial].

## Background

Splat the Net was introduced in the paper "Splat the Net: Radiance Fields with Splattable Neural Primitives" [S1, unofficial]. It addresses the limitations of Gaussian primitives in 3DGS by introducing neural density fields, offering a trade-off between computational cost per primitive and the number of primitives needed. The method extends to dynamic scenes and relighting by incorporating temporal variables and neural color fields [S1, unofficial].

## Sources

[S1] Splat the Net: Radiance Fields with Splattable Neural Primitives, https://doi.org/10.48550/arxiv.2510.08491, fetched 2026-10-26 (unofficial)
