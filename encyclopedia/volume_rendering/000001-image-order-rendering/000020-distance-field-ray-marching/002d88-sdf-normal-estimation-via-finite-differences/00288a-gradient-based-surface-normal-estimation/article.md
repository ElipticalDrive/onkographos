# Gradient-Based Surface Normal Estimation

Also called: CalculateSurfaceNormal, Density Computation & Normal Estimation, Density Field Normal Estimation, Gradient Normal Estimation, Gradient-Based Surface Normal Calculation, Gradient-based normal estimation, Local density field, Normals Renderer

## Overview
Gradient-Based Surface Normal Estimation computes surface normals by evaluating the gradient of a locally computed density field (resembling a signed distance field) on the fly, rather than storing or interpolating explicit normal vectors [S1, S2, unofficial]. It differs from its sibling [Lambertian Shading with Gradient Normals](1229-lambertian-shading-with-gradient-normals.md) by focusing on the general derivation of normals from a voxelized density representation suitable for collision and ray marching, whereas the sibling focuses specifically on the shading application [S1, S2, unofficial]. It avoids the numerical instability of manual finite differencing by using optimized gradient sampling calls where available [S2, unofficial].

## How it works
**Inputs:**
*   A voxelized density field (or signed distance field approximation) stored in a 3D texture or voxel grid.
*   A query point $p$ in world or texture space.
*   A sampling kernel size (typically $2^3$ or small neighborhood) [S1, unofficial].

**Outputs:**
*   A normalized surface normal vector $\mathbf{n}$ at point $p$.

**Data Structures:**
*   **Density Field:** A 3D grid or texture where each voxel stores a scalar density value. In GPU implementations, this is often tightly packed bits representing occupancy or distance, evaluated in parallel [S1, unofficial].
*   **Gradient Vector:** A 3D vector $\mathbf{g} = (g_x, g_y, g_z)$ computed from the density field.

**Steps:**
1.  **Evaluate Density:** Determine the density value at the query point $p$. If using a kernel-based approach, aggregate density from neighboring voxels using a kernel of size $2^3$ (e.g., $2 \times 2 \times 2$ neighborhood) [S1, unofficial].
2.  **Compute Gradient:** Estimate the gradient $\mathbf{g}$ of the density field at $p$.
    *   *Optimized Path:* If the rendering API supports it, call a dedicated gradient sampling function (e.g., `Sampler::SampleGradient(p)`) which returns $\mathbf{g}$ directly [S2, unofficial]. This reduces function calls and potential numerical instability from manual differencing [S2, unofficial].
    *   *Manual Path:* If no dedicated gradient sampler is available, compute the gradient using central finite differences with a step size $\epsilon$:
        $$ g_x = D(p + \epsilon \mathbf{x}) - D(p - \epsilon \mathbf{x}) $$
        $$ g_y = D(p + \epsilon \mathbf{y}) - D(p - \epsilon \mathbf{y}) $$
        $$ g_z = D(p + \epsilon \mathbf{z}) - D(p - \epsilon \mathbf{z}) $$
        where $D$ is the density evaluation function and $\mathbf{x}, \mathbf{y}, \mathbf{z}$ are unit basis vectors [S2, S5, unofficial].
3.  **Normalize:** Compute the surface normal $\mathbf{n}$ by normalizing the gradient vector:
    $$ \mathbf{n} = \frac{\mathbf{g}}{\|\mathbf{g}\|} $$
    [S2, unofficial].
4.  **Usage:** Use $\mathbf{n}$ for shading (e.g., Blinn-Phong specular highlights) or physics (e.g., collision response) [S1, S5, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_normal(p, density_field, epsilon=0.0001):
    """
    Estimate surface normal at point p using gradient of density field.
    
    Args:
        p: Vector3, query point.
        density_field: Function or object providing density evaluation D(p).
        epsilon: Float, step size for finite differences if gradient sampler unavailable.
        
    Returns:
        Vector3, normalized surface normal.
    """
    # Attempt optimized gradient sampling if available [S2, unofficial]
    # This is a placeholder for API-specific call like Sampler::SampleGradient(p)
    try:
        g = density_field.sample_gradient(p)
    except AttributeError:
        # Fallback to manual finite differences [S2, S5, unofficial]
        # Define basis vectors
        x_axis = Vector3(1.0, 0.0, 0.0)
        y_axis = Vector3(0.0, 1.0, 0.0)
        z_axis = Vector3(0.0, 0.0, 1.0)
        
        # Evaluate density at offset points
        d_plus_x = density_field.evaluate(p + epsilon * x_axis)
        d_minus_x = density_field.evaluate(p - epsilon * x_axis)
        d_plus_y = density_field.evaluate(p + epsilon * y_axis)
        d_minus_y = density_field.evaluate(p - epsilon * y_axis)
        d_plus_z = density_field.evaluate(p + epsilon * z_axis)
        d_minus_z = density_field.evaluate(p - epsilon * z_axis)
        
        # Compute gradient components
        g_x = d_plus_x - d_minus_x
        g_y = d_plus_y - d_minus_y
        g_z = d_plus_z - d_minus_z
        
        g = Vector3(g_x, g_y, g_z)
    
    # Normalize gradient to get surface normal [S2, unofficial]
    length = g.length()
    if length < 1e-6:
        return Vector3(0.0, 1.0, 0.0) # Default normal if gradient is zero
    
    return g / length
```

**Parameters:**
*   **Kernel Size:** $2^3$ (8 voxels) is sufficient for acceptable quality in practice [S1, unofficial].
*   **Voxelgrid Resolution:** Commonly $512^3$ for particle demos [S1, unofficial].
*   **Epsilon ($\epsilon$):** $0.0001$ was used in some implementations, but this value is sensitive [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision and Aliasing:** A fixed epsilon (e.g., $0.0001$) may cause precision loss or aliasing if the volume sampling frequency is not aligned with the epsilon scale [S2, unofficial].
*   **GPU Texture Limits:** On DX10 hardware (e.g., G80), 32-bit integer lookup textures require $2^{32}$ entries, which do not fit in GPU memory. Breaking integers into 16-bit parts requires 64 lookups per 1024-bit voxel column, making bitwise arithmetic more efficient than texture lookups for dense voxel storage [S1, unofficial].
*   **Zero Gradient:** If the density field is flat at $p$ (inside or outside the object far from the surface), the gradient magnitude is zero. The normal is undefined; a default or previous frame's normal should be used [G].

## Subtopics

### [SDF Bevel Generation](4312-sdf-bevel-generation.md)
SDF Bevel Generation estimates 3D surface normals by sampling a Signed Distance Function (SDF) height field at four cardinal offsets, computing finite differences, and applying bevel shaping parameters to smooth sharp edges. It differs from standard gradient estimation by introducing explicit control over edge rounding and thickness, making it suitable for stylized or physically plausible beveled geometry in ray marching. It builds on [SDF Normal Estimation via Finite Differences](4619-sdf-normal-estimation-via-finite-differences.md).

### Specular Highlight via Gradient Normal
This variant computes specular highlights by finding the volume surface intersection via ray marching, estimating the surface normal via finite differences of the volume texture, and applying a Blinn-Phong shading model [S5, unofficial]. The normal is computed using central finite differences with a step size `gStepSize` along the x, y, and z axes, then normalized [S5, unofficial]. The specular intensity is calculated as $\text{pow}(\text{dot}(\text{normalize}(\mathbf{H}), \mathbf{N}), 2.0)$, where $\mathbf{H}$ is the halfway vector between light and view directions [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per query, as it involves a fixed number of texture/voxel lookups (typically 6 for central differences or 8 for kernel density) [S1, S2, unofficial].
*   **Memory Complexity:** $O(1)$ additional memory per query, as normals are computed on the fly and not stored [S1, unofficial].
*   **Measured Performance:** In a particle system collision response demo with a $512^3$ voxelgrid, timings were: 0.32 ms for $256^2$ particles, 1.0 ms for $512^2$ particles, and 4.0 ms for $1024^2$ particles [S1, unofficial].
*   **Strengths:**
    *   Does not rely on mesh normals; uses voxelized surface normals for high resolution and coherence [S1, unofficial].
    *   Faster to compute than larger-support distance fields [S1, unofficial].
    *   Simpler to handle than depth peeling for normal estimation [S1, unofficial].
    *   Efficient for applications needing only limited distance information [S1, unofficial].
*   **Limitations:**
    *   Quality depends on voxel resolution; lower resolutions result in blocky normals [S1, unofficial].
    *   Sensitive to epsilon choice in manual finite differencing [S2, unofficial].
    *   GPU implementation details vary significantly between DX9 and DX10 architectures due to texture lookup limitations [S1, unofficial].
*   **When to Use:** When working with voxelized data, particle systems, or dynamic scenes where storing explicit normals is prohibitive, and high-frequency mesh normals are not available [S1, unofficial].
*   **When Not to Use:** When high-precision smooth normals are required from a high-poly mesh (use mesh normals instead) or when the scene is too sparse for a voxel grid to be efficient [S1, unofficial].

## Background
The method was presented in "Single-pass GPU Solid Voxelization and Applications" [S1, unofficial]. It was developed to provide an efficient way to derive normals from solid voxelizations of dynamic scenes at high frame-rates. The approach was shown to outperform previous methods in speed and resolution, leveraging the fact that gradient computation is inexpensive compared to storing explicit normals, especially in the context of large particle systems where the number of queries is much lower than the size of the density map ($512^3$) [S1, unofficial]. The method is compatible with shader-based animation and does not require knowledge of scene geometry beyond depth values [S1, unofficial].

## Sources
[S1] Single-pass GPU Solid Voxelization and Applications, https://inria.hal.science/inria-00345291, fetched 2026-10-26 (unofficial)
[S2] Tsarpf/procgen, https://github.com/Tsarpf/procgen, fetched 2026-10-26 (unofficial)
[S3] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S4] 2JacT/Marching-Cubes-Terrain, https://github.com/2JacT/Marching-Cubes-Terrain, fetched 2026-10-27 (unofficial)
[S5] rbaravalle/webgl-volumetric, https://github.com/rbaravalle/webgl-volumetric, fetched 2026-10-27 (unofficial)
