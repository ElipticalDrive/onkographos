# Exact Rational Ray-Surfel Evaluation

Also called: 2DGS Ray-Surfel Intersection, Homography-based Ray-Surfel Intersection, Screen-space Homography Surfel Intersection

## Overview
Exact Rational Ray-Surfel Evaluation computes the precise intersection point of a viewing ray with a flat surfel rectangle in the surfel's local $(u, v)$ coordinates. It solves the problem of determining where a pixel's ray hits the surfel plane to evaluate the 2D Gaussian opacity, avoiding the per-pixel cross products and divisions required by the original 2DGS formulation [S1, unofficial][S2]. It sets itself apart from [Ray-Splat/Gaussian Intersection](7916-ray-splat-gaussian-intersection.md) by using a homography approximation around the screen-space bounding box center to replace the exact ray-plane intersection with a rational function, significantly reducing computational cost while maintaining visual fidelity [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**:
    *   Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized) [G].
    *   Surfel-to-world homogeneous transformation matrix $H \in \mathbb{R}^{4\times4}$ [S2].
    *   World-to-screen homogeneous transformation matrix $W \in \mathbb{R}^{4\times4}$ [S2].
    *   Pixel coordinates $(x_p, y_p)$ in screen space [S2].
    *   Surfel center $(u_0, v_0)$ in local coordinates [S1, unofficial].
    *   Jacobian matrix $\mathbf{J}$ of the homography approximation [S1, unofficial].
    *   Offset vector $\Delta$ from the bounding box center to the pixel [S1, unofficial].
    *   Ray direction vector $d$ (used in the denominator term) [S1, unofficial].
    *   Base opacity $o_i$ [S2].
*   **Outputs**:
    *   Local intersection coordinates $(u_j, v_j)$ [S2].
    *   Per-pixel opacity $\alpha_i$ [S2].

**Data Structures**
*   **Matrices**: $H$ and $W$ are $4\times4$ homogeneous transformation matrices. $H$ maps local surfel coordinates $(u, v, 1, 1)^\top$ to world space [S2]. $W$ maps world space to screen space [S2].
*   **Vectors**: $\mathbf{h}_x, \mathbf{h}_y, \mathbf{h}_u, \mathbf{h}_v$ are 4D homogeneous plane vectors. $\mathbf{J}$ is a $2\times2$ Jacobian matrix. $\Delta$ is a 2D offset vector.

**Steps**
1.  **Define Screen-Space Planes**: Construct two plane vectors representing the pixel boundaries in screen space:
    $$ \mathbf{h}_x = (-1, 0, 0, x_p)^\top $$
    $$ \mathbf{h}_y = (0, -1, 0, y_p)^\top $$
    [S2]

2.  **Transform to Surfel-Local Frame**: Transform these planes into the surfel's local coordinate system using the combined transformation $WH$. Note that the transformation is applied via the transpose of the inverse (or adjoint) of the matrix product, effectively transforming planes:
    $$ \mathbf{h}_u = (WH)^\top \mathbf{h}_x $$
    $$ \mathbf{h}_v = (WH)^\top \mathbf{h}_y $$
    [S2]

3.  **Compute Intersection Coordinates**: Solve the $2\times2$ linear system derived from the intersection of the ray with the surfel plane defined by $\mathbf{h}_u$ and $\mathbf{h}_v$. Let $h_u^i$ and $h_v^i$ denote the $i$-th component of vectors $\mathbf{h}_u$ and $\mathbf{h}_v$ respectively (1-indexed):
    $$ u_j = \frac{h_u^2 h_v^4 - h_u^4 h_v^2}{h_u^1 h_v^2 - h_u^2 h_v^1} $$
    $$ v_j = \frac{h_u^4 h_v^1 - h_u^1 h_v^4}{h_u^1 h_v^2 - h_u^2 h_v^1} $$
    [S2]

    *Alternative Approximation (Homography)*: For hardware rasterization or performance optimization, the intersection can be approximated using a homography expansion around the center of the screen-space bounding box $(u_0, v_0)$:
    $$ u = u_0 + \frac{(\mathbf{J}\Delta)_u}{1 + d^\top\Delta} $$
    $$ v = v_0 + \frac{(\mathbf{J}\Delta)_v}{1 + d^\top\Delta} $$
    [S1, unofficial]
    Here, $\Delta$ is the offset from the bounding box center, $\mathbf{J}$ is the precomputed Jacobian, and $d$ is the ray direction. This avoids per-pixel cross products and divisions [S1, unofficial].

4.  **Evaluate Opacity**: Compute the per-pixel opacity using the 2D Gaussian falloff in the local frame:
    $$ \alpha_i = o_i \exp\left(-\frac{u_j^2 + v_j^2}{2}\right) $$
    [S2]

**Parameters**
*   **Gaussian Variance**: The exponent uses a variance of 1 (implicit in the formula $-(u^2+v^2)/2$) [S2].
*   **Base Opacity $o_i$**: A scalar parameter per surfel [S2].

**Edge Cases and Pitfalls**
*   **Division by Zero**: In the exact formulation, the denominator $h_u^1 h_v^2 - h_u^2 h_v^1$ represents the determinant of the system. If this is zero, the ray is parallel to the surfel plane or the planes are degenerate [S2].
*   **Precision**: The homography approximation [S1, unofficial] is valid near the center of the bounding box. Far from the center, the rational function may deviate from the exact intersection.
*   **Hardware Rasterization**: When using the homography approximation with hardware rasterization, the comparison with 2DGS was timed only on the CUDA renderer because the hardware rasterizer does not have a native ray-surfel variant [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def exact_ray_surfel_intersection(ray_origin, ray_direction, H, W, x_p, y_p, o_i):
    """
    Computes exact ray-surfel intersection using plane transformation.
    
    Args:
        ray_origin: (3,) array, ray origin.
        ray_direction: (3,) array, normalized ray direction.
        H: (4, 4) array, surfel-to-world homogeneous matrix.
        W: (4, 4) array, world-to-screen homogeneous matrix.
        x_p: float, pixel x coordinate.
        y_p: float, pixel y coordinate.
        o_i: float, base opacity.
        
    Returns:
        alpha: float, per-pixel opacity.
    """
    # Step 1: Define screen-space planes
    h_x = np.array([-1.0, 0.0, 0.0, x_p])
    h_y = np.array([0.0, -1.0, 0.0, y_p])
    
    # Step 2: Transform to surfel-local frame
    # Note: (WH)^T transforms planes from screen to local
    WH = W @ H
    WH_T = WH.T
    
    h_u = WH_T @ h_x
    h_v = WH_T @ h_y
    
    # Step 3: Solve 2x2 system for u_j, v_j
    # Denominator: h_u^1 * h_v^2 - h_u^2 * h_v^1
    # Indices are 1-based in formula, 0-based in code
    denom = h_u[0] * h_v[1] - h_u[1] * h_v[0]
    
    if abs(denom) < 1e-9:
        # Degenerate case: ray parallel to surfel or invalid geometry
        # Return 0 opacity or handle as edge case [G]
        return 0.0
        
    u_j = (h_u[1] * h_v[3] - h_u[3] * h_v[1]) / denom
    v_j = (h_u[3] * h_v[0] - h_u[0] * h_v[3]) / denom
    
    # Step 4: Evaluate opacity
    alpha = o_i * np.exp(-(u_j**2 + v_j**2) / 2.0)
    
    return alpha

def homography_ray_surfel_intersection(u_0, v_0, J, delta, d, o_i):
    """
    Computes approximate ray-surfel intersection using homography.
    
    Args:
        u_0: float, center u coordinate of bounding box.
        v_0: float, center v coordinate of bounding box.
        J: (2, 2) array, Jacobian matrix.
        delta: (2,) array, offset from center.
        d: (3,) array, ray direction (used in denominator term).
        o_i: float, base opacity.
        
    Returns:
        alpha: float, per-pixel opacity.
    """
    # Compute denominator term: 1 + d^T * delta
    # Note: d is 3D, delta is 2D. The source formula uses d^T * delta.
    # Assuming d is projected or delta is padded, or d refers to the 
    # relevant components. Based on [S1, unofficial], d is ray direction.
    # We assume delta is 2D and d's first two components are used, 
    # or d is treated as 2D in this context. 
    # Given the formula structure, we treat d as compatible with delta.
    # [G] Assuming d is truncated to 2D or delta is padded to 3D with 0.
    # Standard homography expansion usually involves 2D ray direction in image space.
    # However, [S1, unofficial] says "d is the ray direction". We will use the first 2 components 
    # if delta is 2D, or pad delta. Let's assume standard 2D homography context.
    
    # Re-reading [S1, unofficial]: "d is the ray direction". 
    # In screen space homography, the ray direction in 3D projects to a 2D vector.
    # Let's assume d here is the 2D direction in the tangent plane or similar.
    # To be safe and consistent with "d^T delta", we dot them.
    
    # If d is 3D and delta is 2D, we need to clarify. 
    # In [S1, unofficial], delta is offset from center (2D). 
    # We will assume d is effectively 2D for this term or delta is padded.
    # Let's pad delta to 3D with 0 for the dot product if d is 3D.
    delta_3d = np.append(delta, 0.0)
    denom_term = 1.0 + np.dot(d, delta_3d)
    
    if abs(denom_term) < 1e-9:
        return 0.0
        
    # Compute J * delta
    J_delta = J @ delta
    
    u = u_0 + J_delta[0] / denom_term
    v = v_0 + J_delta[1] / denom_term
    
    alpha = o_i * np.exp(-(u**2 + v**2) / 2.0)
    
    return alpha
```

## Subtopics

### [Closed-form perspective-correct surfel rasterization](3523-closed-form-perspective-correct-surfel-rasterization.md)
A rasterization technique for planar surfels that computes screen-space texture coordinates $(u, v)$ using an algebraically exact rational function of pixel position, avoiding per-pixel cross products and vector constructions. It sets itself apart from the reference 2DGS formulation by replacing the per-pixel cross-product-based inversion with a precomputed affine approximation, reducing computational cost while maintaining visual fidelity. It is chosen for fragment-bound scenes where the saving per fragment yields significant performance gains (+47% on stump and flowers scenes) compared to scenes bound by sorting and binning (+20% on train scene). The hardware rasterizer has no ray–surfel variant, so performance comparisons were made on the CUDA renderer.

## Operating characteristics
*   **Time Complexity**: The exact method involves matrix-vector multiplications and a $2\times2$ solve, which is constant time $O(1)$ per pixel per surfel. The homography approximation reduces this to a few multiplications and additions, also $O(1)$ but with a smaller constant factor [S1, unofficial][S2].
*   **Memory Complexity**: Requires storing the $4\times4$ matrices $H$ and $W$ per surfel or globally, and per-pixel temporary vectors. Memory usage is low and constant per surfel [S2].
*   **Measured Performance**: On Mip-NeRF 360, Tanks & Temples, and Deep Blending datasets, the hardware rasterizer implementation achieved 2698, 3016, and 4413 FPS respectively, which is 1.8× faster than the CUDA software rasterizer implementation [S1, unofficial]. The method remains 1.1–1.5× faster than Speedy-Splat at lower LPIPS on matched hardware rasterizers [S1, unofficial].
*   **Strengths**: Avoids expensive per-pixel cross products and divisions of the original 2DGS method [S1, unofficial]. Compatible with hardware rasterization for significant speedups [S1, unofficial].
*   **Limitations**: The homography approximation is local to the bounding box center; accuracy may degrade far from the center [S1, unofficial]. The exact method still requires more computation than simple splatting if not optimized [S2].
*   **When to Use**: Use when high-quality surfel rendering is required and performance is critical, especially on hardware rasterizers. Use the exact method when precision is paramount and the homography approximation is insufficient.

## Background
The method originates from 2DGS (2D Gaussian Splatting), which introduced the surfel parameterization and ray-plane intersection for evaluating Gaussian opacity [S2]. Fourier Splatting inherits this ray-surfel intersection method but replaces the fixed Gaussian falloff with Fourier encoded boundaries [S2]. The homography-based approximation was introduced in "Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting" to enable hardware rasterization of surfels by leveraging perspective-correct texturing principles [S1, unofficial].

## Sources
[S1] Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting, https://doi.org/10.48550/arXiv.2607.13808, fetched 2026-10-29 (unofficial)
[S2] Fourier Splatting: Generalized Fourier encoded primitives for scalable radiance fields, https://doi.org/10.48550/arXiv.2603.19834, fetched 2026-10-30
