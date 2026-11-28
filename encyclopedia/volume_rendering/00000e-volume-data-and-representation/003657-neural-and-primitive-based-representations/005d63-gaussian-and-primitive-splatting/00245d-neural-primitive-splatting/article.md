# Neural Primitive Splatting

Also called: Neural Density Field with Splattable Primitives, Neural Primitives, Splattable Neural Primitives, Splatting Neural Primitives

## Overview
Neural Primitive Splatting is a volumetric representation that reconciles the expressivity of neural implicit models with the rendering efficiency of primitive-based splatting [S1, unofficial]. It represents the volumetric kernel itself—the density distribution—as a neural network bounded by an ellipsoid, enabling exact analytical integration along view rays to compute perspectively accurate splatting kernels without costly ray marching [S1, unofficial]. Unlike 3D Gaussian Splatting, which uses an affine approximation of the projection operator, this method yields perspectively accurate results and achieves multi-view consistency by construction, unlike light-field-style approaches [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ primitives $\{P_i\}$, each defined by geometric parameters and a neural network. A camera ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
*   **Output:** A pixel color $\mathbf{C}$ and opacity $\alpha$ computed via front-to-back compositing.

**Data Structures:**
Each primitive $P_i$ is defined by:
1.  **Geometry:** An ellipsoid bounding volume $B$ specified by:
    *   Center $\mathbf{x}_B \in \mathbb{R}^3$
    *   Scaling vector $\mathbf{s}_B \in \mathbb{R}^3$ along principal axes
    *   Rotation quaternion $\mathbf{q}_B \in \mathbb{R}^4$
2.  **Neural Density Field:** A shallow neural network $f_\sigma$ with one hidden layer of width $N_\sigma$.
    *   Weights $W_1 \in \mathbb{R}^{N_\sigma \times 3}$, $W_2 \in \mathbb{R}^{1 \times N_\sigma}$
    *   Biases $\mathbf{b}_1 \in \mathbb{R}^{N_\sigma}$, $b_2 \in \mathbb{R}$
    *   Total parameters per primitive: 99 (with $N_\sigma=8$) [S1, unofficial].

**Numbered Steps:**

1.  **Initialize Network:** Initialize $W_1 \sim U(-1/3, 1/3)$ and $W_2 \sim U(-\sqrt{6/N_\sigma}/\omega_0, \sqrt{6/N_\sigma}/\omega_0)$ [S1, unofficial]. Set hidden width $N_\sigma = 8$ and frequency multiplier $\omega_0 = 30$ [S1, unofficial].

2.  **Ray-Primitive Intersection:** For a view ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, compute the entry distance $t_{in}$ and exit distance $t_{out}$ where the ray intersects the ellipsoid $B$ using analytic line-ellipsoid intersection [S1, unofficial]. If no intersection, the primitive contributes zero.

3.  **Compute Density Integral:** Calculate the accumulated density $\hat{\alpha}$ along the ray segment $[t_{in}, t_{out}]$ using the closed-form antiderivative $S(t; \mathbf{o}, \mathbf{d})$ of the density field $\sigma(\mathbf{x})$ [S1, unofficial].
    $$ \hat{\alpha}(t_{in}, t_{out}, \mathbf{o}, \mathbf{d}) := \int_{t_{in}}^{t_{out}} \sigma(\mathbf{o} + t\mathbf{d}) \, dt = S(t_{out}; \mathbf{o}, \mathbf{d}) - S(t_{in}; \mathbf{o}, \mathbf{d}) $$
    The antiderivative is given by:
    $$ S(t; \mathbf{o}, \mathbf{d}) = [W_2 \oslash (\omega_0 \cdot W_1(\mathbf{d}))] \sin(\omega_0 (t \cdot W_1(\mathbf{d}) + W_1(\mathbf{o}) + \mathbf{b}_1)) + t \cdot b_2 $$
    where $\oslash$ denotes elementwise division [S1, unofficial]. Note that $\sigma(\mathbf{x})$ is never evaluated directly; computations operate on $S$ [S1, unofficial].

4.  **Compute Splatting Kernel Alpha:** Calculate the opacity contribution $\alpha$ for the primitive:
    $$ \alpha(\mathbf{r}) = 1 - \exp(-\max(0, \hat{\alpha}(t_{in}, t_{out}, \mathbf{o}, \mathbf{d}))) $$
    Clamping to zero ensures nonnegative accumulated density [S1, unofficial].

5.  **Compositing:** Sort intersecting primitives by depth. Accumulate color and opacity using standard front-to-back volume rendering compositing [S1, unofficial].

**Parameters:**
*   $N_\sigma = 8$ (Hidden layer width) [S1, unofficial].
*   $\omega_0 = 30$ (Frequency multiplier) [S1, unofficial].
*   Spherical Harmonics bands for color = 4 [S1, unofficial].
*   Training iterations: 100k [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** In the antiderivative formula, $W_1(\mathbf{d})$ could be zero. The sources do not specify a guard for this division. [G] Implementers should add a small epsilon or check for zero vectors in $W_1(\mathbf{d})$ to prevent NaNs.
*   **Normalization:** The density field normalization uses $\|\mathbf{x}_B\|_\infty$ in the denominator in the source text, but the text states normalization is by $\mathbf{x}_B$ and $\mathbf{s}_B$. The formula explicitly writes $\mathbf{x}_B$. [S1, unofficial] This appears to be a typo in the source or an incomplete notation; typically, normalization involves the scale $\mathbf{s}_B$. [G] Implementers should normalize the input coordinate by the ellipsoid's scale $\mathbf{s}_B$ to ensure the network operates on a uniformly scaled domain, as implied by the text description.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def compute_antiderivative(t, o, d, W1, W2, b1, b2, omega0):
    """
    Computes the closed-form antiderivative S(t; o, d) for the neural density field.
    
    Args:
        t: torch.Tensor, scalar or array of t values.
        o: torch.Tensor, ray origin (3,).
        d: torch.Tensor, ray direction (3,).
        W1: torch.Tensor, weight matrix (N_sigma, 3).
        W2: torch.Tensor, weight vector (N_sigma,).
        b1: torch.Tensor, bias vector (N_sigma,).
        b2: torch.Tensor, scalar bias.
        omega0: float, frequency multiplier.
        
    Returns:
        torch.Tensor, antiderivative values.
    """
    # Compute linear term inside sin: omega0 * (t * W1(d) + W1(o) + b1)
    # W1(d) is matrix-vector product: (N_sigma, 3) @ (3,) -> (N_sigma,)
    W1_d = torch.matmul(W1, d)
    W1_o = torch.matmul(W1, o)
    
    linear_term = omega0 * (t * W1_d + W1_o + b1)
    
    # Compute sine
    sin_term = torch.sin(linear_term)
    
    # Compute coefficient: W2 / (omega0 * W1(d))
    # Elementwise division. Guard against zero in denominator.
    denom = omega0 * W1_d
    # [G] Add epsilon to prevent division by zero
    denom = torch.where(denom == 0, torch.tensor(1e-8, device=denom.device), denom)
    coeff = W2 / denom
    
    # Result: coeff * sin(...) + t * b2
    result = coeff * sin_term + t * b2
    
    return result

def render_primitive(ray_o, ray_d, prim_center, prim_scale, prim_quat, W1, W2, b1, b2, omega0):
    """
    Renders a single neural primitive for a given ray.
    
    Args:
        ray_o: torch.Tensor, ray origin (3,).
        ray_d: torch.Tensor, ray direction (3,).
        prim_center: torch.Tensor, ellipsoid center (3,).
        prim_scale: torch.Tensor, ellipsoid scale (3,).
        prim_quat: torch.Tensor, ellipsoid rotation quaternion (4,).
        W1, W2, b1, b2: torch.Tensor, network weights.
        omega0: float, frequency multiplier.
        
    Returns:
        alpha: float, opacity contribution.
    """
    # 1. Analytic line-ellipsoid intersection
    # Transform ray to ellipsoid local space
    # Rotation matrix from quaternion
    R = quat_to_rot_matrix(prim_quat)
    
    # Ray in local space
    o_local = R.T @ (ray_o - prim_center)
    d_local = R.T @ ray_d
    
    # Ellipsoid equation: (x/sx)^2 + (y/sy)^2 + (z/sz)^2 = 1
    # Ray: o_local + t * d_local
    # Solve quadratic for t
    # a = d_local^T S^-2 d_local
    # b = 2 * o_local^T S^-2 d_local
    # c = o_local^T S^-2 o_local - 1
    S_inv = 1.0 / prim_scale
    S_inv2 = S_inv * S_inv
    
    a = torch.dot(d_local, S_inv2 * d_local)
    b = 2.0 * torch.dot(o_local, S_inv2 * d_local)
    c = torch.dot(o_local, S_inv2 * o_local) - 1.0
    
    discriminant = b*b - 4*a*c
    
    if discriminant < 0:
        return 0.0
    
    sqrt_disc = torch.sqrt(discriminant)
    t_in = (-b - sqrt_disc) / (2*a)
    t_out = (-b + sqrt_disc) / (2*a)
    
    # Ensure t_in and t_out are positive (ray direction)
    if t_out < 0:
        return 0.0
    if t_in < 0:
        t_in = t_out # Ray starts inside
        
    # 2. Compute density integral using antiderivative
    # S(t_out) - S(t_in)
    S_out = compute_antiderivative(t_out, ray_o, ray_d, W1, W2, b1, b2, omega0)
    S_in = compute_antiderivative(t_in, ray_o, ray_d, W1, W2, b1, b2, omega0)
    
    alpha_hat = S_out - S_in
    
    # 3. Compute splatting kernel alpha
    # alpha = 1 - exp(-max(0, alpha_hat))
    alpha_hat_clamped = torch.clamp(alpha_hat, min=0.0)
    alpha = 1.0 - torch.exp(-alpha_hat_clamped)
    
    return alpha.item()

def quat_to_rot_matrix(q):
    """Converts quaternion (w, x, y, z) to 3x3 rotation matrix."""
    w, x, y, z = q
    return torch.tensor([
        [1 - 2*y*y - 2*z*z, 2*x*y - 2*w*z, 2*x*z + 2*w*y],
        [2*x*y + 2*w*z, 1 - 2*x*x - 2*z*z, 2*y*z - 2*w*x],
        [2*x*z - 2*w*y, 2*y*z + 2*w*x, 1 - 2*x*x - 2*y*y]
    ])
```

## Operating characteristics
*   **Time Complexity:** Rendering is efficient as it avoids ray marching. The cost is dominated by the analytic line-ellipsoid intersection and the evaluation of the closed-form antiderivative, which involves a few matrix-vector multiplications and trigonometric functions per primitive per ray.
*   **Memory Complexity:** Each primitive uses 99 parameters (with $N_\sigma=8$), which is approximately 1.6× more than Gaussian primitives in 3DGS [S1, unofficial]. However, the method uses 10× fewer primitives and 6× fewer parameters overall to match the quality and speed of 3DGS [S1, unofficial].
*   **Measured Performance:** On the Tanks & Temples dataset, the method achieves PSNR 23.59 vs 23.14 for 3DGS, FPS 158 vs 188 for 3DGS, and Memory 80 MB vs 411 MB for 3DGS [S1, unofficial].
*   **Strengths:** Perspectively accurate results (unlike 3DGS affine approximation), multi-view consistency by construction (unlike light-field approaches), high compactness (fewer primitives/parameters), and real-time efficiency without ray marching.
*   **Limitations:** Slower convergence than Gaussian-based methods due to the complex optimization landscape of neural fields, requiring 100k training iterations [S1, unofficial].
*   **When to use:** Novel-view synthesis on synthetic (Synthetic NeRF) and real datasets (Mip-NeRF360, Tanks & Temples, Deep Blending) where compactness and accuracy are prioritized [S1, unofficial].
*   **When not to use:** When extremely fast training convergence is required, or when the simplest possible primitive representation (pure Gaussians) is sufficient.

## Background
Neural Primitive Splatting was introduced to bridge the gap between neural radiance fields (which are expressive but slow to render) and primitive-based splatting (which is fast but less expressive) [S1, unofficial]. It builds on the principles of Gaussian and Primitive Splatting [17381-gaussian-and-primitive-splatting.md]. The method was developed by the authors of "Splat the Net: Radiance Fields with Splattable Neural Primitives" [S1, unofficial]. It is used for novel-view synthesis tasks [S1, unofficial].

## Sources
[S1] Splat the Net: Radiance Fields with Splattable Neural Primitives, https://doi.org/10.48550/arxiv.2510.08491, fetched 2026-10-26 (unofficial)
