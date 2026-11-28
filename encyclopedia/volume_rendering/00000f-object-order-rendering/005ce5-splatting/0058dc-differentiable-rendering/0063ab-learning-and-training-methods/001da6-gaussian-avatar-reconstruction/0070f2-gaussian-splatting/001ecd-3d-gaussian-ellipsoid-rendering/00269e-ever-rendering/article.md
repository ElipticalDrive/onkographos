# Ever Rendering

Also called: Exact Volumetric Ellipsoid Rendering

## Overview
Ever Rendering is a real-time volumetric rendering method that represents a radiance field as a set of constant-density volumetric ellipsoids, avoiding the 2D billboard approximation of 3D Gaussian Splatting (3DGS) [S1]. Unlike its sibling [Differentiable Ellipsoid Primitive](3490-differentiable-ellipsoid-primitive.md), it replaces rasterization with ray tracing to capture volume density distributions for blending, enabling exact transmittance calculation without approximation errors [S1].

## How it works
### Inputs and Outputs
**Inputs:**
- A set of $N$ 3D Gaussian ellipsoids, each defined by a center position $\mathbf{t}_k$, a rotation matrix $\mathbf{R}_k$, and a scaling vector $\mathbf{s}_k = [s_{kx}, s_{ky}, s_{kz}]^T$ [S1].
- An optimized scalar parameter $\alpha_k \in [0, 1]$ representing the peak opacity along the shortest axis [S1].
- Color $\mathbf{c}_k$ for each ellipsoid, stored as spherical harmonics coefficients for the first two degrees to model view-dependence [S1].
- Camera parameters (position, orientation, intrinsics) [S1].
- Target images for optimization [S1].

**Outputs:**
- A rendered image with color and depth [S1].
- Gradients for optimization via adjoint rendering, which recomputes function values during the backwards pass instead of storing them [S1].

### Data Structures
The scene representation consists of a list of ellipsoids [S1]. Each ellipsoid $k$ stores:
- $\mathbf{t}_k \in \mathbb{R}^3$: Center position.
- $\mathbf{R}_k \in \mathbb{R}^{3 \times 3}$: Rotation matrix (orthonormal).
- $\mathbf{s}_k \in \mathbb{R}^3$: Axis scaling factors.
- $\alpha_k \in \mathbb{R}$: Optimized opacity parameter.
- $\mathbf{c}_k \in \mathbb{R}^3$: Color.
- AABB: Axis-Aligned Bounding Box computed from $\mathbf{t}_k, \mathbf{R}_k, \mathbf{s}_k$ for BVH acceleration [S1].

The acceleration structure is a Binary Volume Hierarchy (BVH) built using NVIDIA OptiX, where each leaf node contains an ellipsoid's AABB and intersection logic [S1].

### Formulas
**Density Calculation:**
To ensure non-zero gradients even when opacity approaches 1, the method optimizes $\alpha_k$ and maps it to density $\sigma(\alpha_k)$ using the minimum axis scale:
$$
\sigma(\alpha_k) = -\frac{\log(1 - 0.99 \cdot \alpha_k)}{\min(s_{kx}, s_{ky}, s_{kz})}
$$
This ensures that a primitive viewed along its shortest axis has a peak opacity equal to $\alpha_k$ [S1].

**Transmittance:**
The volume rendering transmittance is calculated as:
$$
T_i = \prod_{j=1}^{i-1} \exp(-\sigma_j \Delta t_j)
$$
This is identical to the quadrature rule derived by Max [30] using piecewise-constant approximation. In Ever Rendering, this is an intentional design decision with no approximation errors [S1].

**Splitting Condition:**
A splitting condition detects vanished gradients when opacity is near 1 across the entire primitive along the major axis:
$$
0.99 < 1 - \exp(-\sigma(\alpha_k) \cdot \max(s_{kx}, s_{ky}, s_{kz}))
$$
If this inequality holds, the primitive is a candidate for splitting [S1].

**Regularizer Loss:**
To discourage overly anisotropic primitives (which cause large AABBs and slow BVH traversal), a regularizer is applied only to visible primitives:
$$
\mathcal{L}_{reg} = \text{stopgrad}(1 - \alpha) \cdot (\max(s_x, s_y, s_z) - \min(s_x, s_y, s_z))
$$
where $\alpha \in [0, 1]$ is the opacity and $s \in \mathbb{R}^3$ is the ellipsoid axis scaling vector [S1].

### Parameters
- **Splitting Threshold:** $2.5 \times 10^{-7}$ (modified from 3DGS due to smaller spatial gradients) [S1].
- **Cloning Threshold:** $0.1$ [S1].
- **Density Division:** When splitting, the density of the new primitives is divided in half [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def ray_ellipsoid_intersect(ray_o, ray_d, ell_t, ell_R, ell_s):
    """
    Computes near and far intersection distances along the ray.
    Transforms ray to ellipsoid local space and solves quadratic for unit sphere.
    """
    inv_s = 1.0 / ell_s
    local_o = (ell_R.T @ (ray_o - ell_t)) * inv_s
    local_d = (ell_R.T @ ray_d) * inv_s
    
    a = torch.dot(local_d, local_d)
    b = 2.0 * torch.dot(local_o, local_d)
    c = torch.dot(local_o, local_o) - 1.0
    disc = b*b - 4*a*c
    
    if disc < 0:
        return None, None
    
    sqrt_disc = torch.sqrt(disc)
    t1 = (-b - sqrt_disc) / (2 * a)
    t2 = (-b + sqrt_disc) / (2 * a)
    
    if t1 > t2:
        t1, t2 = t2, t1
    return t1, t2

def compute_density(alpha_k, s_k):
    """Maps optimized opacity parameter alpha_k to density sigma."""
    min_scale = torch.min(s_k)
    term = torch.clamp(1.0 - 0.99 * alpha_k, min=1e-7)
    return -torch.log(term) / min_scale

def render_ray(ray_o, ray_d, ellipsoids):
    """
    Renders a single ray using Ever Rendering's front/back event decoupling.
    """
    events = []  # List of (t, delta_sigma, delta_premultiplied_color)
    
    for ell in ellipsoids:
        t_near, t_far = ray_ellipsoid_intersect(ray_o, ray_d, ell.t, ell.R, ell.s)
        if t_near is None:
            continue
            
        sigma = compute_density(ell.alpha, ell.s)
        # Front surface: add density and premultiplied color
        events.append((t_near, sigma, sigma * ell.c))
        # Back surface: subtract density and premultiplied color
        events.append((t_far, -sigma, -sigma * ell.c))
        
    # Sort events by distance along the ray
    events.sort(key=lambda x: x[0])
    
    running_sigma = 0.0
    running_premult_c = torch.zeros(3)
    total_transmittance = 1.0
    accumulated_color = torch.zeros(3)
    prev_t = events[0][0] if events else 0.0
    
    for t, d_sigma, d_premult_c in events:
        delta_t = t - prev_t
        if delta_t > 0 and running_sigma > 0:
            interval_color = running_premult_c / running_sigma
            interval_contrib = interval_color * (1.0 - torch.exp(-running_sigma * delta_t))
            accumulated_color += total_transmittance * interval_contrib
            total_transmittance *= torch.exp(-running_sigma * delta_t)
            if total_transmittance < 1e-4:
                break
        prev_t = t
        running_sigma += d_sigma
        running_premult_c += d_premult_c
        
    return accumulated_color

def optimize_step(ellipsoids, images, targets, bvh):
    """
    One step of optimization with adjoint rendering and ADC.
    """
    rendered_images = []
    for ray in rays:
        color = render_ray(ray.origin, ray.direction, ellipsoids)
        rendered_images.append(color)
        
    loss = torch.mean((torch.stack(rendered_images) - targets) ** 2)
    
    # Add Regularizer Loss
    reg_loss = torch.tensor(0.0)
    for ell in visible_ellipsoids:
        alpha_detached = ell.alpha.detach()
        anisotropy = torch.max(ell.s) - torch.min(ell.s)
        reg_loss += (1.0 - alpha_detached) * anisotropy
        
    loss += reg_loss
    
    # Backward pass (adjoint rendering recomputes forward values)
    loss.backward()
    
    # Adaptive Density Control (ADC)
    for ell in ellipsoids:
        sigma = compute_density(ell.alpha, ell.s)
        max_scale = torch.max(ell.s)
        opacity_major = 1.0 - torch.exp(-sigma * max_scale)
        
        if opacity_major > 0.99:
            split_ellipsoid(ell)
        elif spatial_gradient(ell) > 2.5e-7:
            split_ellipsoid(ell)
        elif spatial_gradient(ell) > 0.1:
            clone_ellipsoid(ell)
            
    return loss

def split_ellipsoid(ell):
    """
    Splits an ellipsoid: reduces size, perturbs position, halves density.
    """
    new_scale = ell.s * 0.5
    perturbation = torch.randn(3) * 0.01
    new_t = ell.t + perturbation
    # Density is halved by adjusting alpha or scale; here we keep alpha and halve contribution
    new_alpha = ell.alpha
    new_c = ell.c / 2.0
    # Create two new primitives with these parameters
    return [Ellipsoid(new_t, ell.R, new_scale, new_alpha, new_c),
            Ellipsoid(new_t, ell.R, new_scale, new_alpha, new_c)]

def clone_ellipsoid(ell):
    """
    Clones an ellipsoid to add detail.
    """
    perturbation = torch.randn(3) * 0.01
    new_t = ell.t + perturbation
    return Ellipsoid(new_t, ell.R, ell.s, ell.alpha, ell.c)

def compute_spatial_gradients(ellipsoids, rendered_images, targets):
    """
    Accumulates xy gradients per primitive from the loss backward pass.
    Standard operation: maps per-pixel gradients to intersected primitives.
    """
    for ell in ellipsoids:
        ell.grad = torch.zeros(3)  # Placeholder for accumulated gradient magnitude
    return ell.grad
```

### Edge Cases and Pitfalls
- **Vanishing Gradients:** If opacity approaches 1, gradients vanish. The mapping $\sigma(\alpha_k)$ ensures non-zero gradients by anchoring $\alpha_k$ to the shortest axis view [S1].
- **Anisotropy:** Highly anisotropic primitives create large AABBs, slowing BVH traversal. The regularizer loss discourages this [S1].
- **Precision:** The log term in density calculation must handle $\alpha_k$ close to 1. The factor $0.99$ prevents $\log(0)$ [S1].
- **BVH Rebuild:** The BVH is rebuilt at every training step, taking tens of milliseconds [S1].

## Operating characteristics
- **Time Complexity:** Real-time rendering is achieved, with performance on Mip-NeRF360 reaching 36 FPS [S1].
- **Memory Complexity:** 1134 MB on Mip-NeRF360 [S1].
- **Performance:** On Mip-NeRF360, EVER achieves PSNR 27.51, SSIM 0.825, LPIPS 0.194 [S1].
- **Scaling:** The method scales to larger scenes (Zip-NeRF) with improved sharpness [S1].
- **Strengths:** Real-time performance, consistent 3D radiance field, no approximation errors in transmittance, improved blending and sharpness compared to 3DGS [S1].
- **Limitations:** BVH rebuild overhead during training (tens of ms per step) [S1].

## Background
Ever Rendering was introduced in the paper "EVER: Exact Volumetric Ellipsoid Rendering for Real-Time View Synthesis" [S1]. It builds on the optimization framework of 3D Gaussian Splatting (3DGS) but replaces rasterization with ray tracing using NVIDIA OptiX and Slang for automatic differentiation [S1]. The method addresses the 2D billboard limitation of 3DGS by using 3D Gaussian ellipsoids that blend into a proper 3D radiance field [S1]. It is used for real-time 3D reconstruction and novel view synthesis, specifically enabling effects like defocus blur and fish-eye camera distortion [S1].

## Sources
[S1] EVER: Exact Volumetric Ellipsoid Rendering for Real-Time View Synthesis, https://doi.org/10.1109/ICCV51701.2025.00469, fetched 2026-10-27
