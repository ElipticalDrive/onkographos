# Semi-transparent Empty-Space Skipping

Also called: Analytic Integration with Empty-Space Skipping, Empty-Space Skipping for Semi-transparent Volumes, Semi-transparent Volumetric Rendering with Empty-Space Skipping

## Overview
Semi-transparent Empty-Space Skipping is an acceleration technique for rendering transparent or semi-transparent volumes that combines empty-space skipping with analytic integration of optical depth. It solves the problem of expensive ray marching inside semi-transparent objects by allowing the ray to skip empty regions outside the volume and then analytically compute the contribution of the segment inside the volume based on its length, rather than sampling it discretely. It sets itself apart from opaque empty-space skipping by using lower bounds for sphere tracing inside objects and from full volumetric rendering by avoiding numerical integration for constant-density materials.

## How it works
The method adapts the BVH-based sphere tracing approach used for opaque volumes to handle transparency. The core distinction lies in how the ray interacts with the volume boundaries and how the optical depth is computed.

### Inputs and Outputs
- **Input**: Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$ (normalized), volume density function $\rho(\mathbf{p})$, transfer function (opacity $\alpha$ per unit length), and a BVH representing the volume's spatial extent.
- **Output**: Final color $C$ and accumulated opacity $\alpha_{acc}$ for the pixel.

### Data Structures
The method relies on a Bounding Volume Hierarchy (BVH). For sphere tracing, each node in the BVH must store bounds that allow for lower-bound distance queries.
- **BVH Node**: Contains bounding volume (e.g., AABB or Sphere) and children indices.
- **Bounds**: For semi-transparent skipping, the bounds must support querying the *minimum* distance from the ray origin to the volume interior, rather than the maximum distance to the surface.

### Steps
1. **Ray Initialization**: Initialize accumulated opacity $\alpha_{acc} = 0$ and color $C = (0,0,0,0)$.
2. **Empty-Space Skipping (Outside)**: While the ray is outside the volume:
   - Perform sphere tracing using the BVH.
   - Use **lower bounds** to determine the distance to the nearest point inside the volume. This allows the ray to quickly advance to the entry point.
   - If the ray misses the volume entirely, terminate and return background color.
3. **Entry Point Detection**: Identify the entry point $\mathbf{p}_{in}$ where the ray intersects the volume boundary.
4. **Inside Volume Processing**:
   - **Case A: Constant Density ($\rho(\mathbf{p}) = \rho_0$)**:
     - Determine the exit point $\mathbf{p}_{out}$ by continuing sphere tracing inside the volume using lower bounds to find the farthest exit or by solving the intersection analytically if the shape is simple.
     - Calculate the segment length $L = \|\mathbf{p}_{out} - \mathbf{p}_{in}\|$.
     - Compute the optical depth $\tau = \rho_0 \cdot L$.
     - Compute the transmittance $T = e^{-\tau}$.
     - Update accumulated opacity: $\alpha_{acc} = 1 - T$.
     - Compute color contribution: $C = C_{in} \cdot (1 - T)$, where $C_{in}$ is the color of the volume material.
     - Terminate ray.
   - **Case B: Varying Density ($\rho(\mathbf{p})$ varies)**:
     - Switch to constant-size ray marching from $\mathbf{p}_{in}$ to $\mathbf{p}_{out}$.
     - Numerically integrate density along the segment: $\tau = \int_{\mathbf{p}_{in}}^{\mathbf{p}_{out}} \rho(\mathbf{p}(t)) dt$.
     - This part is not optimized by the skipping technique and requires full volumetric rendering.
5. **Termination**: If $\alpha_{acc} \ge 1$ or ray length exceeds max distance, terminate.

### Formulas
- **Optical Depth (Constant Density)**:
  $$ \tau = \rho_0 \cdot L $$
  Where $\rho_0$ is the constant density and $L$ is the length of the ray segment inside the volume.
- **Transmittance**:
  $$ T = e^{-\tau} $$
- **Accumulated Opacity**:
  $$ \alpha_{acc} = 1 - T $$

### Parameters
- **Density Function**: $\rho(\mathbf{p})$. Can be constant $\rho_0$ or varying (e.g., FBM noise).
- **Sphere Tracing Step Size**: For varying density, a constant step size $h$ is used for numerical integration. Not given in the sources.
- **BVH Bounds**: Quadratic bounds are recommended for faster convergence away from the entry surface [S1].

### Edge Cases and Pitfalls
- **Varying Density Performance**: Performance gains from empty-space skipping for varying density depend heavily on the amount of empty space in the scene. If the volume is dense with little empty space, the speedup is minimal [S1].
- **Precision**: When using quadratic bounds, ensure numerical stability when calculating distances near the entry surface to avoid jitter or missed intersections [S1].
- **Exit Point Calculation**: For complex shapes, determining the exact exit point $\mathbf{p}_{out}$ may require additional sphere tracing steps or analytical solutions.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_semi_transparent_ray(ray_origin, ray_direction, volume_bvh, density_func, is_constant_density, constant_density_value):
    alpha_acc = 0.0
    color_acc = (0.0, 0.0, 0.0, 0.0)
    t_min = 0.0
    t_max = MAX_DISTANCE

    # Step 1: Skip empty space outside the volume
    t_entry = skip_empty_space(ray_origin, ray_direction, volume_bvh, t_min, t_max)
    
    if t_entry is None:
        return background_color

    p_entry = ray_origin + t_entry * ray_direction

    # Step 2: Determine exit point
    t_exit = find_exit_point(p_entry, ray_direction, volume_bvh, t_entry, t_max)
    
    if t_exit is None:
        return background_color # Should not happen if entry was found

    p_exit = ray_origin + t_exit * ray_direction
    segment_length = length(p_exit - p_entry)

    # Step 3: Compute optical contribution
    if is_constant_density:
        # Analytic integration for constant density
        tau = constant_density_value * segment_length
        transmittance = exp(-tau)
        alpha_acc = 1.0 - transmittance
        color_acc = volume_color * (1.0 - transmittance)
    else:
        # Numerical integration for varying density
        tau = 0.0
        num_steps = int(segment_length / STEP_SIZE)
        for i in range(num_steps):
            t = t_entry + (i + 0.5) * STEP_SIZE / length(ray_direction)
            p = ray_origin + t * ray_direction
            rho = density_func(p)
            tau += rho * STEP_SIZE
        
        transmittance = exp(-tau)
        alpha_acc = 1.0 - transmittance
        color_acc = integrate_color(p_entry, p_exit, density_func, STEP_SIZE) * (1.0 - transmittance)

    return color_acc
```

## Subtopics

### [Object-Order Empty Space Skipping](865-object-order-empty-space-skipping.md)
Renders volumes by traversing the volume data structure directly, skipping empty regions. Choose when cache coherence is critical and the volume is sparse.

### [Occlusion Clipping with Orthogonal Opacity Map](990-occlusion-clipping-with-orthogonal-opacity-map.md)
Uses precomputed opacity maps to clip rays. Choose when multiple views are rendered and precomputation is feasible.

### [Ray Marcher with Macrocell Empty Space Skipping](1009-ray-marcher-with-macrocell-empty-space-skipping.md)
Uses macrocells to skip large empty regions. Choose for very large scenes with hierarchical empty space.

### [Empty Space Skipping with Orthogonal BSP Tree](1054-empty-space-skipping-with-orthogonal-bsp-tree.md)
Uses a BSP tree for empty space skipping. Choose when the volume has a regular grid structure.

### [Ray Casting with Depth/Alpha Test for Empty Space Skipping](1537-ray-casting-with-depth-alpha-test-for-empty-space-skipping.md)
Uses depth and alpha tests to skip rays. Choose for hardware-accelerated rasterization pipelines.

### [Empty Space Skipping with Octree](1610-empty-space-skipping-with-octree.md)
Uses an octree to skip empty regions. Choose for hierarchical volume data.

### [Spatial Skipping (Occupancy Grid)](1614-spatial-skipping-occupancy-grid.md)
Uses an occupancy grid to skip empty voxels. Choose for simple, uniform grids.

### [Parallel Tree-Construction for Empty-Space Skipping](5037-parallel-tree-construction-for-empty-space-skipping.md)
Parallel construction of BVHs for empty space skipping. Choose for large-scale parallel rendering.

### [Transfer-Function Independent Empty-Space Skipping](5038-transfer-function-independent-empty-space-skipping.md)
Skips empty space independent of the transfer function. Choose when multiple transfer functions are used.

## Operating characteristics
- **Time Complexity**: Depends on the BVH traversal cost and the number of samples inside the volume. For constant density, it is $O(\log N)$ for skipping plus $O(1)$ for analytic integration. For varying density, it is $O(\log N)$ for skipping plus $O(M)$ for ray marching, where $M$ is the number of steps inside the volume.
- **Memory Complexity**: $O(N)$ for the BVH, where $N$ is the number of nodes.
- **Measured Performance**: On a Clouds scene with constant density, the 'Ours, Impr Quad.' method achieves a 5.7x speedup over naive Sphere tracing (5.6ms vs 31.9ms), reducing average octaves from 1892.0 to 196.3 [S1]. On the same scene with varying density, it achieves a 1.8x speedup (37.9ms vs 66.5ms), reducing average octaves from 4087.4 to 1675.9 [S1].
- **Strengths**: Significant speedup for constant density volumes; efficient empty-space skipping for varying density volumes.
- **Limitations**: Performance gains for varying density depend on the amount of empty space; requires accurate exit point calculation.
- **When to Use**: Ideal for semi-transparent volumes with constant density or sparse varying density volumes.
- **When Not to Use**: Not suitable for dense varying density volumes with little empty space, as the overhead of BVH traversal may not be justified.

## Background
The method was introduced in "Fast sphere tracing of procedural volumetric noise for very large and detailed scenes" [S1]. It builds on the concept of using BVHs for sphere tracing in opaque volumes and extends it to semi-transparent cases by incorporating analytic integration for constant density materials. The technique leverages quadratic bounds to improve convergence speed near the entry surface [S1].

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
