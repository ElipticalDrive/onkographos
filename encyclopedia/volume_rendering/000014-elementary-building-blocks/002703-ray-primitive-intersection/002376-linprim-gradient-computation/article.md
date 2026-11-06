# LinPrim Gradient Computation

Also called: Backpropagation through ray space intersection, Gradient propagation for LinPrim

## Overview
LinPrim Gradient Computation computes the gradients of intersection depths $\mathbf{i}$ with respect to the ray-space triangle corners $\mathbf{v}^{\text{ray}}$ to enable differentiable rendering of linear primitives [S1]. It differs from standard 3D Gaussian Splatting (3DGS) by backpropagating gradients through a linear primitive rasterizer (MTIA) rather than a Gaussian splatting kernel, requiring specific handling of ray-space to world-space transformations and quaternion-based rotation updates [S1]. It builds on [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md) by replacing the Gaussian alpha-blending gradient flow with opacity calculations based on intersection depths [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Gradients of the loss with respect to the final pixel color $\frac{\partial L}{\partial C}$, the intersection depths $\mathbf{i}_1, \mathbf{i}_2$ for the primitive, the ray-space triangle corners $\mathbf{v}_0^{\text{ray}}, \mathbf{v}_1^{\text{ray}}, \mathbf{v}_2^{\text{ray}}$, and the barycentric coordinates $u, v$ of the intersection point [S1].
*   **Output:** Gradients of the loss with respect to the ray-space corners $\frac{\partial L}{\partial \mathbf{v}_k^{\text{ray}}}$, which are subsequently transformed to world space and propagated to primitive parameters (distance, rotation) [S1].

**Data Structures:**
*   **Ray-space corners:** $\mathbf{v}_k^{\text{ray}} \in \mathbb{R}^3$ for $k \in \{0, 1, 2\}$ [S1].
*   **Intersection depths:** $\mathbf{i}_1, \mathbf{i}_2 \in \mathbb{R}$, representing the entry and exit depths of the ray through the primitive along the view ray [S1].
*   **Barycentric coordinates:** $u, v \in [0, 1]$ such that $u+v \le 1$ [S1].

**Step-by-Step Method:**

1.  **Opacity Gradient Calculation:**
    Compute the gradient of the loss with respect to the intersection depths $\mathbf{i}_1$ and $\mathbf{i}_2$ using the opacity function [S1]:
    $$ o(\mathbf{i}_1, \mathbf{i}_2) = 1 - \exp(-\sigma(\alpha) \cdot (\mathbf{i}_2 - \mathbf{i}_1)) $$
    where $\sigma(\alpha)$ is a scaling factor for opacity $\alpha$ [S1]. The gradient $\frac{\partial L}{\partial \mathbf{i}_k}$ is derived from $\frac{\partial L}{\partial C}$ and $\frac{\partial o}{\partial \mathbf{i}_k}$ [S1].

2.  **Barycentric Coordinate Derivatives:**
    Compute the partial derivatives of the barycentric coordinates $u$ and $v$ with respect to the ray-space corners [S1]. The barycentric coordinates are invariant to depth, so they have only two non-zero components (x and y) [S1]. Let $\mathbf{r}$ be the pixel ray coordinates and $d$ be the determinant of the triangle projection in ray space [S1]. For corner $\mathbf{v}_0^{\text{ray}}$ [S1]:
    $$ \frac{\partial u}{\partial \mathbf{v}_0^{\text{ray}}} = \frac{1}{d} \left( \begin{pmatrix} \mathbf{v}_{2,y}^{\text{ray}} - \mathbf{r}_y \\ \mathbf{r}_x - \mathbf{v}_{2,x}^{\text{ray}} \end{pmatrix} - u \begin{pmatrix} \mathbf{v}_{2,y}^{\text{ray}} - \mathbf{v}_{1,y}^{\text{ray}} \\ \mathbf{v}_{1,x}^{\text{ray}} - \mathbf{v}_{2,x}^{\text{ray}} \end{pmatrix} \right) $$
    $$ \frac{\partial v}{\partial \mathbf{v}_0^{\text{ray}}} = \frac{1}{d} \left( \begin{pmatrix} \mathbf{r}_y - \mathbf{v}_{1,y}^{\text{ray}} \\ \mathbf{v}_{1,x}^{\text{ray}} - \mathbf{r}_x \end{pmatrix} - v \begin{pmatrix} \mathbf{v}_{2,y}^{\text{ray}} - \mathbf{v}_{1,y}^{\text{ray}} \\ \mathbf{v}_{1,x}^{\text{ray}} - \mathbf{v}_{2,x}^{\text{ray}} \end{pmatrix} \right) $$
    Analogous formulas apply for $\mathbf{v}_1^{\text{ray}}$ and $\mathbf{v}_2^{\text{ray}}$ [S1].

3.  **Intersection Depth Gradient w.r.t. Corners:**
    Compute the gradient of the intersection depth $\mathbf{i}$ with respect to each corner $\mathbf{v}_k^{\text{ray}}$ [S1]. The source provides specific formulas for $\mathbf{v}_0^{\text{ray}}$ [S1]:
    $$ \frac{\partial \mathbf{i}}{\partial \mathbf{v}_0^{\text{ray}}} = \begin{pmatrix} -\left( \frac{\partial u}{\partial \mathbf{v}_{0,y}^{\text{ray}}} + \frac{\partial v}{\partial \mathbf{v}_{0,x}^{\text{ray}}} \right) \cdot \mathbf{v}_{0,z}^{\text{ray}} + \frac{\partial u}{\partial \mathbf{v}_{0,x}^{\text{ray}}} \cdot \mathbf{v}_{1,z}^{\text{ray}} + \frac{\partial v}{\partial \mathbf{v}_{0,x}^{\text{ray}}} \cdot \mathbf{v}_{2,z}^{\text{ray}} \\ - \left( \frac{\partial u}{\partial \mathbf{v}_{0,y}^{\text{ray}}} + \frac{\partial v}{\partial \mathbf{v}_{0,y}^{\text{ray}}} \right) \cdot \mathbf{v}_{0,z}^{\text{ray}} + \frac{\partial u}{\partial \mathbf{v}_{0,y}^{\text{ray}}} \cdot \mathbf{v}_{1,z}^{\text{ray}} + \frac{\partial v}{\partial \mathbf{v}_{0,y}^{\text{ray}}} \cdot \mathbf{v}_{2,z}^{\text{ray}} \\ 1 - u - v \end{pmatrix} $$
    The source text contains typos in indices for $\mathbf{v}_1^{\text{ray}}$ and $\mathbf{v}_2^{\text{ray}}$ (e.g., $\partial u / \partial \mathbf{v}_{1,x}$ appearing twice) [S1]. Implementers should derive analogous formulas based on symmetry [S1]. Corrected symmetric formulas for $\mathbf{v}_1^{\text{ray}}$ and $\mathbf{v}_2^{\text{ray}}$ are:
    $$ \frac{\partial \mathbf{i}}{\partial \mathbf{v}_1^{\text{ray}}} = \begin{pmatrix} -\left( \frac{\partial u}{\partial \mathbf{v}_{1,y}^{\text{ray}}} + \frac{\partial v}{\partial \mathbf{v}_{1,x}^{\text{ray}}} \right) \cdot \mathbf{v}_{1,z}^{\text{ray}} + \frac{\partial u}{\partial \mathbf{v}_{1,x}^{\text{ray}}} \cdot \mathbf{v}_{0,z}^{\text{ray}} + \frac{\partial v}{\partial \mathbf{v}_{1,x}^{\text{ray}}} \cdot \mathbf{v}_{2,z}^{\text{ray}} \\ - \left( \frac{\partial u}{\partial \mathbf{v}_{1,y}^{\text{ray}}} + \frac{\partial v}{\partial \mathbf{v}_{1,y}^{\text{ray}}} \right) \cdot \mathbf{v}_{1,z}^{\text{ray}} + \frac{\partial u}{\partial \mathbf{v}_{1,y}^{\text{ray}}} \cdot \mathbf{v}_{0,z}^{\text{ray}} + \frac{\partial v}{\partial \mathbf{v}_{1,y}^{\text{ray}}} \cdot \mathbf{v}_{2,z}^{\text{ray}} \\ -v \end{pmatrix} $$
    $$ \frac{\partial \mathbf{i}}{\partial \mathbf{v}_2^{\text{ray}}} = \begin{pmatrix} -\left( \frac{\partial u}{\partial \mathbf{v}_{2,y}^{\text{ray}}} + \frac{\partial v}{\partial \mathbf{v}_{2,x}^{\text{ray}}} \right) \cdot \mathbf{v}_{2,z}^{\text{ray}} + \frac{\partial u}{\partial \mathbf{v}_{2,x}^{\text{ray}}} \cdot \mathbf{v}_{0,z}^{\text{ray}} + \frac{\partial v}{\partial \mathbf{v}_{2,x}^{\text{ray}}} \cdot \mathbf{v}_{1,z}^{\text{ray}} \\ - \left( \frac{\partial u}{\partial \mathbf{v}_{2,y}^{\text{ray}}} + \frac{\partial v}{\partial \mathbf{v}_{2,y}^{\text{ray}}} \right) \cdot \mathbf{v}_{2,z}^{\text{ray}} + \frac{\partial u}{\partial \mathbf{v}_{2,y}^{\text{ray}}} \cdot \mathbf{v}_{0,z}^{\text{ray}} + \frac{\partial v}{\partial \mathbf{v}_{2,y}^{\text{ray}}} \cdot \mathbf{v}_{1,z}^{\text{ray}} \\ -u \end{pmatrix} $$

4.  **Ray-Space Center Gradient:**
    Compute the gradient with respect to the ray-space center [S1]. The process follows the same procedure as for the corners [S1].

5.  **Transformation to World Space:**
    Undo the ray-space and view-space transformations [S1]. Since these are matrix multiplications, the gradients are propagated by multiplying with the transpose of the transformation matrices [G].

6.  **Propagation to Primitive Parameters:**
    Map the world-space corner gradients to the primitive's distance and rotation parameters [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_linprim_gradients(loss_grad_color, i1, i2, v0_ray, v1_ray, v2_ray, r, sigma_alpha):
    """
    Computes gradients for LinPrim intersection depths.
    
    Args:
        loss_grad_color: Gradient of loss w.r.t. pixel color.
        i1, i2: Intersection depths (entry, exit).
        v0_ray, v1_ray, v2_ray: Ray-space triangle corners (3D vectors).
        r: Pixel ray coordinates (2D vector).
        sigma_alpha: Scaling factor for opacity.
        
    Returns:
        grad_v0, grad_v1, grad_v2: Gradients w.r.t. ray-space corners.
    """
    # Step 1: Opacity Gradient
    exp_term = np.exp(-sigma_alpha * (i2 - i1))
    grad_i2 = loss_grad_color * sigma_alpha * exp_term
    grad_i1 = -loss_grad_color * sigma_alpha * exp_term
    
    # Step 2: Barycentric Derivatives
    dx1 = v1_ray[0] - v0_ray[0]
    dy1 = v1_ray[1] - v0_ray[1]
    dx2 = v2_ray[0] - v0_ray[0]
    dy2 = v2_ray[1] - v0_ray[1]
    d = dx1 * dy2 - dy1 * dx2
    
    # Compute barycentric coordinates u, v for intersection point r
    u = ((v2_ray[1] - r[1]) * (v0_ray[0] - r[0]) + (r[1] - v0_ray[1]) * (v2_ray[0] - r[0])) / d
    v = ((v0_ray[1] - r[1]) * (v1_ray[0] - r[0]) + (r[1] - v0_ray[1]) * (v0_ray[0] - r[0])) / d
    
    # Helper to compute 2D barycentric derivatives for a corner
    def barycentric_derivs(v_k, v_next, v_prev):
        # v_k is the corner being differentiated, v_next is the next corner in CCW order, v_prev is previous
        # Derivatives w.r.t v_k.x and v_k.y
        du_dk = np.array([v_prev[1] - r[1], r[0] - v_prev[0]]) / d - u * np.array([v_prev[1] - v_next[1], v_next[0] - v_prev[0]]) / d
        dv_dk = np.array([r[1] - v_next[1], v_next[0] - r[0]]) / d - v * np.array([v_prev[1] - v_next[1], v_next[0] - v_prev[0]]) / d
        return du_dk, dv_dk

    du_v0, dv_v0 = barycentric_derivs(v0_ray, v1_ray, v2_ray)
    du_v1, dv_v1 = barycentric_derivs(v1_ray, v2_ray, v0_ray)
    du_v2, dv_v2 = barycentric_derivs(v2_ray, v0_ray, v1_ray)
    
    # Step 3: Intersection Depth Gradient w.r.t. Corners
    def compute_intersection_grad(du_k, dv_k, v_k, v_next, v_prev, u_val, v_val):
        # 2D components
        grad_x = -((du_k[1] + dv_k[0]) * v_k[2] + du_k[0] * v_prev[2] + dv_k[0] * v_next[2])
        grad_y = -((du_k[1] + dv_k[1]) * v_k[2] + du_k[1] * v_prev[2] + dv_k[1] * v_next[2])
        # 3D component (barycentric coordinate derivative)
        grad_z = 1.0 - u_val - v_val
        return np.array([grad_x, grad_y, grad_z])

    grad_i_v0 = compute_intersection_grad(du_v0, dv_v0, v0_ray, v1_ray, v2_ray, u, v)
    grad_i_v1 = compute_intersection_grad(du_v1, dv_v1, v1_ray, v2_ray, v0_ray, u, v)
    grad_i_v1[2] = -v_val  # Corrected z-component for v1
    grad_i_v2 = compute_intersection_grad(du_v2, dv_v2, v2_ray, v0_ray, v1_ray, u, v)
    grad_i_v2[2] = -u_val  # Corrected z-component for v2
    
    # Step 4 & 5: Backpropagate through opacity and transform to world space
    grad_v0_ray = grad_i_v0 * (grad_i1 - grad_i2)
    grad_v1_ray = grad_i_v1 * (grad_i1 - grad_i2)
    grad_v2_ray = grad_i_v2 * (grad_i1 - grad_i2)
    
    # World space transformation (transpose of view/ray matrices) [G]
    # grad_v0_world = T_view.T @ T_ray.T @ grad_v0_ray
    # grad_v1_world = T_view.T @ T_ray.T @ grad_v1_ray
    # grad_v2_world = T_view.T @ T_ray.T @ grad_v2_ray
    
    return grad_v0_ray, grad_v1_ray, grad_v2_ray
```

**Parameters:**
*   **Color learning rate:** $2.5 \times 10^{-3}$ [S1].
*   **SH Coefficients learning rate:** $1.25 \times 10^{-4}$ [S1].
*   **Opacity learning rate:** $2.5 \times 10^{-2}$ [S1].
*   **Rotation learning rate:** $1 \times 10^{-3}$ [S1].
*   **Distance learning rate:** $2.6 \times 10^{-4}$, scaled by the maximum distance between a known camera pose and the mean camera pose [S1].
*   **2D filter kernel size:** $0.1$ [S1]. For octahedra, the ray space vector is moved by half that amount ($0.05$) [S1].

**Edge Cases and Pitfalls:**
*   **Octahedra Negation:** For octahedra, vertices are in positive or negative directions from the center [S1]. The gradient onto the actual ray-space feature can be negated [S1]. This negation is omitted in the provided formulas for readability [S1].
*   **Density Normalization:** Do not propagate gradients through the density normalization factor [S1].
*   **Source Typos:** The source text contains typos in indices for $\mathbf{v}_1$ and $\mathbf{v}_2$ gradients (e.g., $\partial u / \partial \mathbf{v}_{1,x}$ appearing twice) [S1]. Implementers must correct these based on symmetry [S1].

## Subtopics

### [LinPrim Optimization](3492-linprim-optimization.md)
LinPrim Optimization is the parameter update phase of the LinPrim rendering pipeline, which adjusts the geometric and appearance features of linear primitives (triangles) to minimize a differentiable loss function. It differs from standard 3D Gaussian Splatting (3DGS) optimization by backpropagating gradients through a linear primitive rasterizer (MTIA) rather than a Gaussian splatting kernel, requiring specific handling of ray-space to world-space transformations and quaternion-based rotation updates. It builds on [LinPrim Gradient Computation](3038-linprim-gradient-computation.md) to compute the necessary gradients for intersection depths.

## Operating characteristics
*   **Time Complexity:** Removing the ray space approximation increases rendering time from 15.87ms to 29.15ms on ScanNet++ scene 39f36da05b, solely due to rasterization increasing from 14.60ms to 28.34ms [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On ScanNet++ scene 39f36da05b, LinPrim + MCMC achieves PSNR 27.62, SSIM 0.858, LPIPS 0.272 [S1].
*   **Scaling:** The distance learning rate is scaled by the maximum distance between a known camera pose and the mean camera pose to ensure consistent behavior in scale-ambiguous settings [S1].
*   **Strengths:** LinPrim is robust to initialization perturbations, with minimal performance drop on ScanNet++ 39f36da05b when using noisy or half points [S1].

## Background
*   **Origin:** LinPrim was introduced in the paper "LinPrim: Linear Primitives for Differentiable Volumetric Rendering" [S1].
*   **History:** The method extends differentiable rendering to linear primitives, using a ray-space approximation to accelerate rasterization [S1].
*   **Usage:** Used in differentiable volumetric rendering for 3D reconstruction [S1].

## Sources
[S1] LinPrim: Linear Primitives for Differentiable Volumetric Rendering, https://arxiv.org/abs/2501.16312, fetched 2026-10-26
