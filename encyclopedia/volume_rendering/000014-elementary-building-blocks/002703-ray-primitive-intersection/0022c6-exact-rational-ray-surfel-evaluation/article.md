# Exact Rational Ray-Surfel Evaluation

Also called: Homography-based Ray-Surfel Intersection, Screen-space Homography Surfel Intersection

## Overview
Exact Rational Ray-Surfel Evaluation computes the intersection of a viewing ray with a flat surfel rectangle in its local $(u, v)$ coordinates using a homography approximation around the center of the screen-space bounding box [S1, unofficial]. It sets itself apart from the reference 2DGS formulation by replacing the per-pixel cross-product-based inversion with a precomputed affine approximation, reducing computational cost while maintaining visual fidelity [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$ (normalized), surfel center in world space $\mathbf{c}$, surfel local basis vectors $\mathbf{u}_{vec}, \mathbf{v}_{vec}$, surfel center in local coordinates $(u_0, v_0)$, Jacobian matrix $\mathbf{J}$, and screen-space offset $\Delta$ from the bounding box center to the current pixel.
*   **Output:** Local coordinates $(u, v)$ of the intersection point on the surfel.

**Data Structures**
The method relies on precomputed constants for each surfel:
*   $\mathbf{J}$: A $2 \times 2$ Jacobian matrix representing the linear approximation of the homography.
*   $(u_0, v_0)$: The local coordinates of the center of the surfel's screen-space bounding box.
*   $\mathbf{c}$: The world-space center of the surfel.

**Steps**
1.  Compute the vector $\Delta$ from the center of the surfel's screen-space bounding box to the current pixel's screen-space position.
2.  Compute the dot product $d^{\top}\Delta$. Note: In the source formula, $d$ represents the ray direction projected or transformed appropriately to align with the screen-space offset context, or $\Delta$ is defined in a space compatible with $\mathbf{d}$ [S1, unofficial]. Specifically, the denominator is $1 + d^{\top}\Delta$.
3.  Compute the numerator components using the Jacobian $\mathbf{J}$ and offset $\Delta$:
    $$ (\mathbf{J}\Delta)_u = \mathbf{J}_{0,0}\Delta_x + \mathbf{J}_{0,1}\Delta_y $$
    $$ (\mathbf{J}\Delta)_v = \mathbf{J}_{1,0}\Delta_x + \mathbf{J}_{1,1}\Delta_y $$
4.  Calculate the local coordinates:
    $$ u = u_0 + \frac{(\mathbf{J}\Delta)_u}{1 + d^{\top}\Delta} $$
    $$ v = v_0 + \frac{(\mathbf{J}\Delta)_v}{1 + d^{\top}\Delta} $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def exact_rational_ray_surfel_intersection(ray_origin, ray_direction, surfel_center, surfel_u_vec, surfel_v_vec, 
                                          u0, v0, J, delta):
    """
    Computes local (u, v) coordinates of the ray-surfel intersection.
    
    Args:
        ray_origin: (3,) array, origin of the viewing ray.
        ray_direction: (3,) array, normalized direction of the viewing ray.
        surfel_center: (3,) array, world-space center of the surfel.
        surfel_u_vec: (3,) array, local u-axis vector of the surfel.
        surfel_v_vec: (3,) array, local v-axis vector of the surfel.
        u0: float, local u-coordinate of the bounding box center.
        v0: float, local v-coordinate of the bounding box center.
        J: (2, 2) array, Jacobian matrix of the homography approximation.
        delta: (2,) array, screen-space offset from bounding box center to pixel.
        
    Returns:
        u: float, local u-coordinate of intersection.
        v: float, local v-coordinate of intersection.
    """
    # Compute the denominator term
    # Note: The source formula uses d^T * delta. 
    # In a strict implementation, 'd' in this context must be compatible with 'delta'.
    # If delta is screen-space, d might be the projected ray direction or a specific vector 
    # derived from the ray and surfel geometry. The source implies a direct dot product.
    # We assume 'd' here refers to the ray direction vector as provided, 
    # but typically this requires projection or alignment. 
    # Based on the formula provided:
    
    # Calculate dot product d^T * delta
    # If delta is 2D and d is 3D, this operation is undefined without projection.
    # However, the source formula is explicit: 1 + d^T * delta.
    # We assume the inputs are pre-processed such that dimensions match or d is the relevant 2D component.
    # For the purpose of this implementation based strictly on the formula:
    
    # Let's assume 'd' passed is the relevant vector for the dot product with delta.
    # If ray_direction is 3D and delta is 2D, we need to project ray_direction to screen space 
    # or delta is in a space where ray_direction is 2D. 
    # Given the "Exact Rational" nature, this is likely a specific algebraic simplification.
    # We will implement the formula as stated, assuming compatible dimensions.
    
    d_dot_delta = np.dot(ray_direction[:2], delta) # Assuming 2D projection or specific d
    
    denominator = 1.0 + d_dot_delta
    
    # Calculate Jacobian * delta
    J_delta = np.dot(J, delta)
    
    # Calculate u and v
    u = u0 + J_delta[0] / denominator
    v = v0 + J_delta[1] / denominator
    
    return u, v
```

**Parameters**
*   **Jacobian $\mathbf{J}$**: Precomputed for each surfel based on its orientation and position relative to the camera.
*   **$(u_0, v_0)$**: Center of the screen-space bounding box of the surfel.

**Edge Cases and Pitfalls**
*   **Denominator Zero**: If $1 + d^{\top}\Delta = 0$, the division is undefined. This corresponds to a singularity in the homography approximation.
*   **Precision**: The method avoids per-pixel cross products and divisions, improving performance, but relies on the accuracy of the homography approximation around the bounding box center.
*   **Boundaries**: The approximation is valid around the center of the screen-space bounding box. Large bounding boxes may reduce accuracy.

## Subtopics

### [Closed-form perspective-correct surfel rasterization](3523-closed-form-perspective-correct-surfel-rasterization.md)
A rasterization technique for planar surfels that computes screen-space texture coordinates $(u, v)$ using an algebraically exact rational function of pixel position, avoiding per-pixel cross products and vector constructions. It sets itself apart from the reference 2DGS formulation by replacing the per-pixel cross-product-based inversion with a precomputed affine approximation, reducing computational cost while maintaining visual fidelity.

## Operating characteristics
**Time and Memory Complexity**
The method reduces the per-pixel computational cost by moving most calculations (Jacobian, center coordinates) to precomputation. The per-pixel cost is reduced to a few dot products and a division.

**Measured Performance**
*   On Mip-NeRF 360, Tanks & Temples, and Deep Blending, the hardware rasterizer implementation achieved mean frame rates of 2698, 3016, and 4413 FPS, respectively, which is 1.8× over the CUDA software rasterizer implementation [S1, unofficial].
*   Fragment-bound scenes gain the most performance (+47% on stump and flowers scenes), while scenes bound by sorting and binning gain the least (+20% on train scene) [S1, unofficial].
*   The hardware rasterizer generates fragments only for pixels covered by the surfels, requiring significantly less memory and computation than tile-based binning [S1, unofficial].

**Strengths and Limitations**
*   **Strengths**: High performance due to reduced per-pixel operations and efficient use of hardware rasterization. Maintains visual quality with minimal deviation (at most 0.05 dB and 0.0007 LPIPS) compared to software rasterization [S1, unofficial].
*   **Limitations**: The approximation is local to the bounding box center. The hardware rasterizer does not have a native ray-surfel variant, so the intersection comparison was timed only on the CUDA renderer for some metrics [S1, unofficial].

**When to Use**
Use this method when performance is critical and the surfels are relatively small in screen space, ensuring the homography approximation remains accurate. It is particularly effective for fragment-bound scenes.

## Background
This method was introduced in "Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting" by Niedermayr et al. (2024) [S1, unofficial]. It addresses the high per-pixel cost of ray-surfel intersection in 2D Gaussian Splatting (2DGS) by leveraging the homography property of planar surfaces in perspective projection.

## Sources
[S1] Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting, https://doi.org/10.48550/arXiv.2607.13808, fetched 2026-10-29 (unofficial)
