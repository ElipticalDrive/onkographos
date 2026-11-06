# Splat the Net

Also called: Neural Primitives, Splat the Net: Radiance Fields with Splattable Neural Primitives

## Overview
Splat the Net is a neural primitive-based volume rendering method that replaces the Gaussian kernels of 3D Gaussian Splatting (3DGS) with splattable neural primitives [S1, unofficial]. It represents 3D scenes using a set of parameterized primitives, each containing a small Multi-Layer Perceptron (MLP) for density and Spherical Harmonics (SH) for color, allowing for higher expressivity per primitive [S1, unofficial]. The method achieves comparable real-time rendering speeds (>100 FPS) to 3DGS while requiring approximately 10 times fewer primitives for the same scene representation [S1, unofficial]. It builds on [Neural and primitive-based representations](10-neural-and-primitive-based-representations.md) by introducing neural density fields into the splatting framework.

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of $N$ primitives, each defined by geometric parameters and neural weights.
*   Camera parameters (position, orientation, intrinsics).
*   For dynamic scenes: a temporal variable $\xi_t \in [0, 1]$.
*   For relighting: a light direction $\mathbf{d}_l$.

**Outputs:**
*   Rendered RGB images for given camera viewpoints.

### Data Structures
Each primitive $i$ stores the following parameters, totaling 99 parameters per primitive (compared to 59 for 3DGS) [S1, unofficial]:

1.  **Geometry (10 parameters):**
    *   Mean position $\mathbf{\mu} \in \mathbb{R}^3$.
    *   Scale $\mathbf{s} \in \mathbb{R}^3$.
    *   Rotation represented as a unit quaternion $\mathbf{q} \in \mathbb{R}^4$.
2.  **Neural Density MLP (41 parameters):**
    *   An 8-neuron MLP with specific weights and biases (details in formulas below).
3.  **Color Representation (48 parameters):**
    *   Zero-order Spherical Harmonics (SH) coefficients. For dynamic scenes, these are time-dependent.

### Formulas

**1. Neural Density Field**
The density $\sigma$ at a 3D position $\mathbf{x}$ is computed by a neural primitive. For static scenes, the density is derived from an MLP. For dynamic scenes, an Eulerian approach incorporates time $\xi_t$ [S1, unofficial]:

$$
f_*(\mathbf{x}, \xi_t) = W_2 \cos(\omega_0 \cdot W_1(\mathbf{x}) + \xi_t \cdot W_t + \mathbf{b}_1) + \mathbf{b}_2
$$

Where:
*   $W_1, W_2, \mathbf{b}_1, \mathbf{b}_2$ are learnable weights and biases of the MLP.
*   $\omega_0$ is a frequency parameter.
*   $W_t \in \mathbb{R}^{N_\sigma}$ is a learnable temporal weight vector.
*   $\xi_t \in [0, 1]$ is the normalized timestamp.

**2. Dynamic Color (Time-Dependent SH)**
The zero-order SH coefficient $S$ for a primitive is modeled as a function of time $\xi_t$, combining a polynomial and a Fourier series [S1, unofficial]:

$$
S(\xi_t) = S_0 + P_n(\xi_t) + F_l(\xi_t)
$$

Where:
*   $S_0$ is the base zero-order SH coefficient.
*   $P_n(\xi_t) = \sum_{i=1}^n a_i \xi_t^i$ is a polynomial of degree $n$.
*   $F_l(\xi_t)$ is a Fourier series of order $l$.
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

def compute_density_neural_primitive(x, params, t=None):
    """
    Computes density using the neural density field.
    x: (3,) spatial coordinate
    params: dict containing W1, W2, b1, b2, omega0, Wt (if dynamic)
    t: (1,) temporal variable xi_t, None for static
    """
    # Static part
    z = np.dot(params['W1'], x) + params['b1']
    if t is not None:
        z = z + np.dot(params['Wt'], t)
    
    # Activation
    z = np.cos(params['omega0'] * z)
    
    # Output
    sigma = np.dot(params['W2'], z) + params['b2']
    return sigma

def compute_color_dynamic_sh(sh_coeffs, t, n=4, l=4):
    """
    Computes time-dependent SH coefficient.
    sh_coeffs: (N,) base SH coefficients S0
    t: (1,) temporal variable
    """
    S0 = sh_coeffs
    
    # Polynomial part
    P_n = 0
    for i in range(1, n + 1):
        # ai are learned parameters, assumed to be in sh_coeffs or separate
        # Assuming ai are stored in a separate array 'poly_coeffs'
        # For this pseudocode, we assume poly_coeffs is available
        P_n += poly_coeffs[i-1] * (t ** i)
        
    # Fourier part
    F_l = 0
    for k in range(1, l + 1):
        # bk, ck are learned parameters
        F_l += bk[k-1] * np.cos(k * t) + ck[k-1] * np.sin(k * t)
        
    return S0 + P_n + F_l

def render_ray(ray_origin, ray_direction, primitives, camera_params, t=None):
    """
    Renders a single ray using splatting.
    """
    # Sort primitives by depth
    sorted_primitives = sort_primitives_by_depth(primitives, ray_origin, ray_direction)
    
    alpha_accum = 0.0
    color_accum = np.array([0.0, 0.0, 0.0])
    
    for prim in sorted_primitives:
        # Compute distance from ray to primitive center
        # Simplified: assume primitive is a point with density field
        # In reality, this involves projecting the primitive to 2D and evaluating
        # the density field at the intersection or closest point
        
        # Evaluate density at a point along the ray within the primitive's influence
        # This is a simplified representation of the splatting process
        sigma = compute_density_neural_primitive(prim.center, prim.mlp_params, t)
        
        # Compute opacity alpha from sigma and primitive size
        alpha = 1.0 - np.exp(-sigma * prim.scale_factor)
        
        if alpha < 1e-5:
            continue
            
        # Compute color
        if t is not None:
            color = compute_color_dynamic_sh(prim.sh_coeffs, t)
        else:
            color = prim.sh_coeffs # Static case, direct SH coeff
            
        # Alpha compositing
        weight = alpha * (1.0 - alpha_accum)
        color_accum += weight * color
        alpha_accum += weight
        
        if alpha_accum > 0.99:
            break
            
    return color_accum, alpha_accum

def train_step(primitives, images, targets, optimizer, iteration, dataset_type):
    """
    One training step including population control.
    """
    # Forward pass
    rendered_images = []
    for img in images:
        # Render all pixels
        rendered_img = render_image(img, primitives)
        rendered_images.append(rendered_img)
        
    # Compute loss
    loss = compute_loss(rendered_images, targets)
    
    # Backward pass
    gradients = compute_gradients(loss, primitives)
    optimizer.step(gradients)
    
    # Population Control
    if should_densify(iteration, dataset_type):
        densify_primitives(primitives, gradients, dataset_type)
    if should_prune(primitives, dataset_type):
        prune_primitives(primitives, dataset_type)
        
    return loss

def should_densify(iteration, dataset_type):
    if dataset_type == 'dynamic':
        return 1000 <= iteration <= 30000 and iteration % 500 == 0
    elif dataset_type in ['mipnerf360', 'tanks', 'deep_blending']:
        return 1000 <= iteration <= 15000 and iteration % 500 == 0
    elif dataset_type == 'blendernerf':
        return 1000 <= iteration <= 20000 and iteration % 200 == 0
    return False

def densify_primitives(primitives, gradients, dataset_type):
    densify_threshold = 1e-4 if dataset_type != 'blendernerf' else 1e-5
    grow_threshold = 1e-2
    
    for prim in primitives:
        grad_mag = np.linalg.norm(gradients[prim.id])
        if grad_mag > densify_threshold and prim.scale < grow_threshold:
            # Split or add new primitive
            add_new_primitive(prim)

def prune_primitives(primitives, dataset_type):
    prune_threshold = 0.5
    
    for prim in primitives:
        if prim.scale > prune_threshold:
            remove_primitive(prim)
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
