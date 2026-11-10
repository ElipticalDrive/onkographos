# Neural Primitive Splatting

Also called: Neural Density Field with Splattable Primitives, Neural Primitives, Splattable Neural Primitives, Splatting Neural Primitives

## Overview
Neural Primitive Splatting is a volumetric representation that reconciles the expressivity of neural implicit fields with the rendering efficiency of primitive-based splatting [S1, unofficial]. It defines a scene as a mixture of bounded volumetric primitives, where each primitive contains a shallow neural network parameterizing a spatially varying density field [S1, unofficial]. Unlike sibling methods that splat Gaussian functions with neural augmentations, this method represents the volumetric kernel (density distribution) itself as a neural network, enabling exact analytical integration along view rays [S1, unofficial]. This allows for perspectively accurate splatting kernels without the computational cost of ray marching, while using significantly fewer primitives and parameters than 3D Gaussian Splatting (3DGS) to achieve comparable quality [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ neural primitives $\{P_i\}_{i=1}^N$, each defined by geometric parameters and neural weights. A camera ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the origin and $\mathbf{d}$ is the direction.
*   **Output:** A pixel color $\mathbf{C}$ and accumulated opacity $\alpha$ via front-to-back compositing.

**Data Structures:**
Each primitive $P_i$ is defined by:
1.  **Bounding Ellipsoid $B$:**
    *   Center $\mathbf{x}_B \in \mathbb{R}^3$
    *   Scaling vector $\mathbf{s}_B \in \mathbb{R}^3$ (along principal axes)
    *   Rotation quaternion $\mathbf{q}_B \in \mathbb{R}^4$
2.  **Neural Density Field Weights:**
    *   Weight matrix $W_1 \in \mathbb{R}^{N_\sigma \times 3}$
    *   Weight matrix $W_2 \in \mathbb{R}^{1 \times N_\sigma}$
    *   Bias vector $\mathbf{b}_1 \in \mathbb{R}^{N_\sigma}$
    *   Bias scalar $b_2 \in \mathbb{R}$
    *   Total parameters per primitive: 99 (with $N_\sigma=8$) [S1, unofficial].

**Numbered Steps:**

1.  **Primitive Initialization:**
    Initialize weights for each primitive:
    *   $W_1 \sim U(-1/3, 1/3)$
    *   $W_2 \sim U(-\sqrt{6/N_\sigma}/\omega_0, \sqrt{6/N_\sigma}/\omega_0)$ [S1, unofficial].
    *   Set hidden layer width $N_\sigma = 8$ and frequency multiplier $\omega_0 = 30$ [S1, unofficial].

2.  **Ray-Primitive Intersection:**
    For a view ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, compute the entry distance $t_{in}$ and exit distance $t_{out}$ where the ray intersects the bounding ellipsoid $B$ of primitive $P_i$. This is done via analytic line-ellipsoid intersection [S1, unofficial].

3.  **Compute Density Integral (Antiderivative):**
    Evaluate the closed-form antiderivative $S(t; \mathbf{o}, \mathbf{d})$ of the density function along the ray. The density field $\sigma(\mathbf{x})$ is defined within the ellipsoid $B$ as:
    $$ \sigma(\mathbf{x}) = f_\sigma\left(\frac{\mathbf{x} - \mathbf{x}_B}{\|\mathbf{x}_B\|_\infty}\right) $$
    where $f_\sigma$ is the neural network [S1, unofficial]. Note: The source text writes $\|\mathbf{x}_B\|_\infty$ in the denominator, but context implies normalization by the ellipsoid bounds (likely involving $\mathbf{s}_B$) to ensure $f_\sigma$ operates on a centered and uniformly scaled domain [S1, unofficial].

    The neural network activation is:
    $$ f_\sigma(\mathbf{x}) = W_2 \left( \cos \left( \omega_0 \left( W_1(\mathbf{x}) + \mathbf{b}_1 \right) \right) \right) + b_2 $$
    where $\cos$ is applied element-wise [S1, unofficial].

    The antiderivative $S(t; \mathbf{o}, \mathbf{d})$ with respect to $t$ is:
    $$ S(t; \mathbf{o}, \mathbf{d}) = [W_2 \oslash (\omega_0 \cdot W_1(\mathbf{d}))] \sin(\omega_0 (t \cdot W_1(\mathbf{d}) + W_1(\mathbf{o}) + \mathbf{b}_1)) + t \cdot b_2 $$
    Here, $\oslash$ denotes element-wise division, and $\cdot$ denotes matrix-vector multiplication [S1, unofficial].

4.  **Compute Accumulated Density:**
    Calculate the integral of density along the ray segment within the ellipsoid:
    $$ \hat{\alpha}(t_{in}, t_{out}, \mathbf{o}, \mathbf{d}) = S(t_{out}; \mathbf{o}, \mathbf{d}) - S(t_{in}; \mathbf{o}, \mathbf{d}) $$
    [S1, unofficial].

5.  **Compute Splatting Kernel Alpha:**
    Compute the opacity contribution $\alpha$ for the primitive:
    $$ \alpha(\mathbf{r}) = 1 - \exp(-\max(0, \hat{\alpha}(t_{in}, t_{out}, \mathbf{o}, \mathbf{d}))) $$
    Clamping to zero ensures non-negative accumulated density [S1, unofficial].

6.  **Color Computation:**
    The color associated with the primitive is typically represented using Spherical Harmonics (SH) coefficients (4 bands used) [S1, unofficial]. The view-dependent color $\mathbf{c}$ is evaluated based on the ray direction $\mathbf{d}$ and the primitive's SH coefficients.

7.  **Compositing:**
    Sort all intersected primitives by depth ($t_{in}$). Perform front-to-back compositing to determine the final pixel color $\mathbf{C}$ and opacity $\alpha$ using the standard volume rendering equation [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def compute_antiderivative(t, o, d, W1, W2, b1, b2, omega0):
    """
    Computes S(t; o, d) for a single primitive.
    o: ray origin (3,)
    d: ray direction (3,)
    W1: weight matrix (N_sigma, 3)
    W2: weight matrix (1, N_sigma)
    b1: bias vector (N_sigma,)
    b2: bias scalar
    omega0: frequency multiplier
    """
    # W1(d) is a vector of size N_sigma
    W1_d = np.dot(W1, d)
    # W1(o) is a vector of size N_sigma
    W1_o = np.dot(W1, o)
    
    # Argument for sin: omega0 * (t * W1(d) + W1(o) + b1)
    # t is scalar, W1_d is vector
    arg = omega0 * (t * W1_d + W1_o + b1)
    
    # sin(arg) is vector of size N_sigma
    sin_val = np.sin(arg)
    
    # W2 is (1, N_sigma), sin_val is (N_sigma,)
    # Element-wise division: W2 / (omega0 * W1(d))
    # Note: W2 is (1, N_sigma), W1_d is (N_sigma,)
    # Broadcasting: W2 / (omega0 * W1_d) results in (1, N_sigma)
    divisor = omega0 * W1_d
    # Avoid division by zero if necessary, though W1_d is unlikely to be 0 everywhere
    safe_divisor = np.where(np.abs(divisor) > 1e-8, divisor, 1e-8)
    coeff = W2 / safe_divisor
    
    # Term 1: coeff * sin_val -> (1, N_sigma) * (N_sigma,) -> (1,)
    term1 = np.dot(coeff, sin_val)
    
    # Term 2: t * b2 -> scalar
    term2 = t * b2
    
    return term1 + term2

def render_primitive(primitive, ray_o, ray_d):
    """
    Renders a single primitive for a given ray.
    primitive: dict with keys 'x_B', 's_B', 'q_B', 'W1', 'W2', 'b1', 'b2', 'SH_coeffs'
    ray_o: ray origin (3,)
    ray_d: ray direction (3,)
    """
    # 1. Compute t_in and t_out via analytic line-ellipsoid intersection
    # This requires transforming the ray into the ellipsoid's local space
    # Ellipsoid equation in local space: x'^2/s_x^2 + y'^2/s_y^2 + z'^2/s_z^2 = 1
    # Transform ray: o' = R^T * (o - x_B), d' = R^T * d
    # Solve quadratic for t
    # ... (Implementation of line-ellipsoid intersection omitted for brevity, assumed available)
    t_in, t_out = line_ellipsoid_intersection(primitive, ray_o, ray_d)
    
    if t_in > t_out:
        return 0.0, np.zeros(3) # No intersection

    # 2. Compute Antiderivative at t_out and t_in
    S_out = compute_antiderivative(t_out, ray_o, ray_d, 
                                   primitive['W1'], primitive['W2'], 
                                   primitive['b1'], primitive['b2'], 
                                   primitive['omega0'])
    S_in = compute_antiderivative(t_in, ray_o, ray_d, 
                                  primitive['W1'], primitive['W2'], 
                                  primitive['b1'], primitive['b2'], 
                                  primitive['omega0'])
    
    # 3. Compute accumulated density
    alpha_hat = S_out - S_in
    
    # 4. Compute kernel alpha
    alpha = 1.0 - math.exp(-max(0.0, alpha_hat))
    
    if alpha < 1e-6:
        return 0.0, np.zeros(3)
        
    # 5. Compute color using SH
    # ... (SH evaluation based on ray_d and primitive['SH_coeffs'])
    color = evaluate_sh(primitive['SH_coeffs'], ray_d)
    
    return alpha, color

def render_scene(primitives, ray_o, ray_d):
    """
    Renders a scene by compositing primitives.
    """
    # 1. Find all intersected primitives and compute t_in
    intersections = []
    for p in primitives:
        t_in, t_out = line_ellipsoid_intersection(p, ray_o, ray_d)
        if t_in < t_out:
            intersections.append((t_in, p))
            
    # 2. Sort by depth
    intersections.sort(key=lambda x: x[0])
    
    # 3. Front-to-back compositing
    accumulated_alpha = 0.0
    final_color = np.zeros(3)
    
    for t_in, p in intersections:
        alpha, color = render_primitive(p, ray_o, ray_d)
        
        # Standard volume rendering compositing
        # C = C_prev + alpha * color * (1 - accumulated_alpha)
        # alpha_total = alpha_total + alpha * (1 - accumulated_alpha)
        
        weight = alpha * (1.0 - accumulated_alpha)
        final_color += weight * color
        accumulated_alpha += weight
        
        if accumulated_alpha > 0.99:
            break
            
    return final_color, accumulated_alpha
```

**Parameters:**
*   Hidden layer width $N_\sigma = 8$ [S1, unofficial].
*   Frequency multiplier $\omega_0 = 30$ [S1, unofficial].
*   Spherical Harmonics bands for color: 4 [S1, unofficial].
*   Training iterations: 100k [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** In the antiderivative formula, element-wise division by $\omega_0 \cdot W_1(\mathbf{d})$ occurs. If $W_1(\mathbf{d})$ is close to zero, numerical instability may arise. A small epsilon should be used for safe division [G].
*   **Normalization:** The density normalization uses $\|\mathbf{x}_B\|_\infty$ in the source text, but context suggests it should involve the scaling vector $\mathbf{s}_B$ to properly center and scale the domain for $f_\sigma$ [S1, unofficial]. Implementers should verify the correct normalization factor.
*   **Gradient-based Densification:** Unlike 3DGS, which uses screen-space gradients, this method uses the gradient magnitude of the network weights for population control (densification/pruning) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering is efficient due to closed-form integration, avoiding ray marching. The cost is dominated by the number of primitives and the per-primitive arithmetic operations for the antiderivative.
*   **Memory Complexity:** Memory usage is proportional to the number of primitives. Each primitive uses 99 parameters (weights) plus geometric parameters. This is ~1.6x more than a 3DGS primitive, but the method uses 10x fewer primitives and 6x fewer total parameters [S1, unofficial].
*   **Measured Performance:**
    *   On Tanks & Temples: PSNR 23.59 (vs 23.14 for 3DGS), FPS 158 (vs 188 for 3DGS), Memory 80 MB (vs 411 MB for 3DGS) [S1, unofficial].
    *   Matches the quality and speed of 3DGS while using significantly fewer resources [S1, unofficial].
*   **Strengths:**
    *   Perspectively accurate splatting (unlike 3DGS's affine approximation) [S1, unofficial].
    *   Multi-view consistency by construction (unlike light-field-style approaches) [S1, unofficial].
    *   High expressivity of neural density fields with real-time rendering efficiency [S1, unofficial].
*   **Limitations:**
    *   Slower convergence during training (100k iterations) due to the complex optimization landscape of neural fields [S1, unofficial].
    *   Requires analytic line-ellipsoid intersection implementation.

## Background
Neural Primitive Splatting was introduced to bridge the gap between neural radiance fields (which are expressive but slow to render) and primitive-based splatting (which is fast but less expressive) [S1, unofficial]. It builds on the concept of shallow neural networks with periodic activations, which allow for closed-form integration [S1, unofficial]. The method was developed to address the limitations of prior splatting methods that only augmented Gaussian primitives with neural features, rather than representing the density kernel itself as a neural network [S1, unofficial]. It is used for novel-view synthesis on synthetic datasets (Synthetic NeRF) and real datasets (Mip-NeRF360, Tanks & Temples, Deep Blending) [S1, unofficial].

## Sources
[S1] Splat the Net: Radiance Fields with Splattable Neural Primitives, https://doi.org/10.48550/arxiv.2510.08491, fetched 2026-10-26 (unofficial)
