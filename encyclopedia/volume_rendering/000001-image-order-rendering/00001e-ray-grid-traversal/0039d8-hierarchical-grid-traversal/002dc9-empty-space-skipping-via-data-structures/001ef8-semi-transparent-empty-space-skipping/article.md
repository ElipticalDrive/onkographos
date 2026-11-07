# Semi-transparent Empty-Space Skipping

Also called: Analytic Integration with Empty-Space Skipping, Empty-Space Skipping for Semi-transparent Volumes, Semi-transparent Volumetric Rendering with Empty-Space Skipping

## Overview
Semi-transparent Empty-Space Skipping is a rendering technique for transparent or semi-transparent volumes that combines hierarchical acceleration structures with analytical integration to skip inactive regions. Unlike opaque volume rendering which stops at the first hit, this method allows rays to traverse through semi-transparent media, evaluating the optical contribution of the traversed segment analytically based on its length and density. It distinguishes itself by using lower bounds in bounding volume hierarchies (BVH) for sphere tracing inside objects, enabling efficient empty-space skipping for both constant and varying density materials [S1].

## How it works
The method operates in two distinct modes depending on the density function $\rho(p)$ of the volume: constant density and varying density. It relies on a BVH acceleration structure where each node stores bounds for the density field.

### Inputs and Outputs
*   **Input:** Ray origin $\mathbf{o}$ and direction $\mathbf{d}$; Volume density function $\rho(p)$; Transfer function (for opacity); BVH structure with density bounds.
*   **Output:** Final color and accumulated opacity for the ray.

### Data Structures
The acceleration structure is a BVH. Each node stores:
*   **Bounds:** Axis-aligned bounding box (AABB) or sphere enclosing the children.
*   **Density Bounds:** For semi-transparent rendering, nodes store lower bounds ($\rho_{min}$) and potentially upper bounds ($\rho_{max}$) of the density within the region. For constant density, $\rho_{min} = \rho_{max} = \rho$ [S1].

### Steps

1.  **Ray Initialization:** Start ray at $\mathbf{o}$. Initialize accumulated opacity $\alpha_{acc} = 0$ and color $C_{acc} = 0$.
2.  **BVH Query:** Query the BVH to find the nearest intersection with an active region.
    *   For **constant density** materials, use sphere tracing with **lower bounds**. If a node's lower bound is zero (or below a threshold), the ray can skip the entire node. If the lower bound is non-zero, the ray must enter the region [S1].
    *   For **varying density** materials, use sphere tracing with the proposed BVH methods to perform fast empty-space skipping outside the volume. Inside the volume, full volumetric rendering is required [S1].
3.  **Traversal and Integration:**
    *   **Case A: Constant Density $\rho(p) = \rho_0$**
        *   When the ray enters a region with density $\rho_0$, it continues toward the exit point.
        *   The optical depth $\tau$ for a segment of length $L$ is calculated analytically: $\tau = \rho_0 \cdot L$.
        *   The transmittance is $T = e^{-\tau}$.
        *   The contribution of the segment is added to the accumulator. The ray is advanced to the exit point of the current active region.
        *   Quadratic bounds can be used to quickly move away from the entry surface, improving performance [S1].
    *   **Case B: Varying Density $\rho(p)$**
        *   **Outside Volume:** Use sphere tracing with the BVH to skip empty space. The ray advances by the distance to the nearest active region boundary.
        *   **Inside Volume:** Switch to constant-size ray marching. Numerically integrate the density along the ray segment within the active region. This part is not optimized by the skipping technique itself, as the density varies [S1].
4.  **Termination:** Stop when the ray exits the volume or accumulated opacity $\alpha_{acc}$ exceeds a threshold (e.g., 1.0).

### Formulas
*   **Optical Depth (Constant Density):**
    $$ \tau = \int_{0}^{L} \rho_0 \, ds = \rho_0 \cdot L $$
*   **Transmittance:**
    $$ T = e^{-\tau} $$
*   **Color Accumulation:**
    $$ C_{out} = C_{in} \cdot T + C_{local} \cdot (1 - T) $$
    Where $C_{local}$ is the emitted/scattered color of the segment.

### Parameters
*   **Density Threshold:** For varying density, a threshold determines if a region is "active". Not given in the sources.
*   **Ray Marching Step Size:** For varying density inside volumes, a constant step size is used. Not given in the sources.

### Pitfalls
*   **Performance Dependency:** For varying density, performance gains depend heavily on the amount of empty space in the scene. If the volume is dense, the skipping benefit is reduced [S1].
*   **Precision:** Using lower bounds for sphere tracing requires careful handling of floating-point precision to avoid missing thin features or incorrectly skipping active regions [S1].
*   **Boundary Handling:** Correctly identifying entry and exit points for constant density regions is critical for analytical integration.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_semi_transparent_ray(ray_origin, ray_direction, bvh, density_func, is_constant_density, constant_density_value):
    t = 0.0
    acc_alpha = 0.0
    acc_color = [0.0, 0.0, 0.0]
    
    while acc_alpha < 1.0:
        # Query BVH for next active region
        hit = bvh.query(ray_origin + ray_direction * t, ray_direction)
        
        if hit is None:
            break
            
        entry_t = hit.entry_t
        exit_t = hit.exit_t
        
        # Skip empty space
        if entry_t > t:
            t = entry_t
            
        segment_length = exit_t - t
        
        if is_constant_density:
            # Analytical integration for constant density
            optical_depth = constant_density_value * segment_length
            transmittance = exp(-optical_depth)
            
            # Calculate color contribution (assuming emission E)
            # C_local = E * (1 - T)
            # acc_color = acc_color * T + C_local
            # This simplifies to: acc_color = acc_color * T + E * (1 - T)
            # But standard volume rendering: C_out = C_in * T + E * (1 - T)
            
            # Update accumulator
            acc_color = acc_color * transmittance + [1.0, 1.0, 1.0] * (1.0 - transmittance) # Example emission
            acc_alpha = 1.0 - transmittance * (1.0 - acc_alpha) # Simplified opacity update
            
            # Advance ray to exit
            t = exit_t
        else:
            # Varying density: Ray march inside the region
            # This part is not optimized by skipping
            step_size = 0.01 # Not given in sources
            num_steps = int(segment_length / step_size)
            
            for i in range(num_steps):
                current_t = t + i * step_size
                pos = ray_origin + ray_direction * current_t
                density = density_func(pos)
                
                # Numerical integration
                optical_depth = density * step_size
                transmittance = exp(-optical_depth)
                
                acc_color = acc_color * transmittance + [1.0, 1.0, 1.0] * (1.0 - transmittance)
                acc_alpha = 1.0 - transmittance * (1.0 - acc_alpha)
                
            t = exit_t
            
    return acc_color, acc_alpha
```

## Operating characteristics
*   **Time Complexity:** Depends on the BVH traversal and the density of the volume. For constant density, it is significantly faster than naive sphere tracing due to analytical integration and skipping. For varying density, it is faster than naive sphere tracing outside the volume, but inside the volume, it performs full ray marching [S1].
*   **Memory Complexity:** Depends on the BVH size. Not given in the sources.
*   **Performance:** On a Clouds scene with constant density, the method achieves a 5.7x speedup over naive sphere tracing (5.6ms vs 31.9ms), reducing average octaves from 1892.0 to 196.3. For varying density, it achieves a 1.8x speedup (37.9ms vs 66.5ms), reducing average octaves from 4087.4 to 1675.9 [S1].
*   **Strengths:** Efficient for constant density semi-transparent volumes; effective empty-space skipping for varying density volumes outside the active region.
*   **Limitations:** For varying density, the benefit is limited to the empty space outside the volume; inside the volume, full ray marching is required. Performance is sensitive to the amount of empty space [S1].

## Background
The method is described in the paper "Fast sphere tracing of procedural volumetric noise for very large and detailed scenes" [S1]. It extends opaque sphere tracing techniques to semi-transparent volumes by using lower bounds in BVHs for constant density materials and combining skipping with ray marching for varying density materials. The approach highlights the use of quadratic bounds to improve performance by quickly moving away from entry surfaces [S1].

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
