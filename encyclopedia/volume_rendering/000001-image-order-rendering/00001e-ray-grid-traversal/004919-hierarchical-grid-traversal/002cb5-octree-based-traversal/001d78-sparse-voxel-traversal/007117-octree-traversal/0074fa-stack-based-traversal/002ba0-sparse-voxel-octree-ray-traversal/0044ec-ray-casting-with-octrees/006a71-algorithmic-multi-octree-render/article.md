# Algorithmic Multi-Octree Render

Also called: Ray-Interval Sorting, Segmented Volume Rendering, Compositional Voxel Rendering, Macro-level ray-interval sorting

## Overview
Algorithmic Multi-Octree Render is a rendering strategy for dynamic scenes that decouples a static background from independently moving foreground assets using a multi-octree representation [S1, unofficial]. It solves the problem of maintaining correct front-to-back occlusion and transmittance propagation when dynamic objects change coordinates arbitrarily, which would otherwise require costly runtime spatial tree reconstruction [S1, unofficial]. Unlike baseline methods that rely on a single strictly ordered front-to-back sequence via Morton ordering, this method uses explicit ray-box intersections to pre-order ray intervals, allowing independent moving objects to be processed in their local canonical frames while guaranteeing mathematically correct multi-volume occlusion [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Static background octree $O_{bg}$, a set of $N$ dynamic foreground asset octrees $\{O_i\}_{i=1}^N$, transformation matrices $\{T_t^i\}_{i=1}^N$ for each asset at timestep $t$, a camera ray $r(\tau) = o + \tau d$, and a pre-sorted sequence of intervals $I$ derived from ray-box intersections [S1, unofficial].
*   **Outputs:** Final pixel color $C$, depth $D$, normal $N$, and residual transmittance $T$ [S1, unofficial].

**Data Structures:**
*   **State Tuple $S$:** A tuple $(C, D, N, T, \tau_{curr})$ tracking the global pixel state along the ray path [S1, unofficial].
    *   $C$: Accumulated color.
    *   $D$: Rendered surface depth.
    *   $N$: Aggregated voxel normals.
    *   $T$: Residual transmittance.
    *   $\tau_{curr}$: Active ray distance boundary [S1, unofficial].
*   **Intervals $I$:** A sequence of $[\tau_{in}^i, \tau_{out}^i]$ pairs sorted by camera distance, representing the entry and exit points of dynamic assets along the ray [S1, unofficial].

**Parameters:**
*   $N$: Number of independently moving foreground asset octrees [S1, unofficial].
*   $\epsilon_{stop}$: Threshold for residual transmittance $T$ below which rendering stops [S1, unofficial].
*   $\tau_{max}$: Maximum ray distance for the remaining background render [S1, unofficial].
*   $C_{sky}$: View-dependent color of the infinite sky background [S1, unofficial].

**Algorithm:**
The method processes each ray by iterating through the sorted intervals. It distinguishes between "Background Segments" (outside dynamic bounding boxes) and "Asset Segments" (inside dynamic bounding boxes) [S1, unofficial].

1.  Initialize the state tuple $S$ with $C=0, D=0, N=0, T=1$ and set $\tau_{curr} = 0$ [S1, unofficial].
2.  Iterate through each dynamic asset $i$ from $1$ to $N$:
    a.  **Render Background:** Process the background octree $O_{bg}$ for the interval $[\tau_{curr}, \tau_{in}^i]$ using the accumulation function $F$. Update state $S \leftarrow F(O_{bg}, r, [\tau_{curr}, \tau_{in}^i], S)$ [S1, unofficial].
    b.  **Transform Ray:** Transform the camera ray $r$ into the local canonical space of the current object using the inverse transformation: $r_{local} \leftarrow (T_t^i)^{-1} (r)$ [S1, unofficial].
    c.  **Render Asset:** Process the asset octree $O_i$ for the interval $[\tau_{in}^i, \tau_{out}^i]$ using the local ray $r_{local}$ and the accumulation function $F$. Update state $S \leftarrow F(O_i, r_{local}, [\tau_{in}^i, \tau_{out}^i], S)$ [S1, unofficial].
    d.  **Advance Boundary:** Update the current ray distance boundary: $\tau_{curr} \leftarrow \tau_{out}^i$ [S1, unofficial].
    e.  **Early Exit:** If $T < \epsilon_{stop}$, break the loop [S1, unofficial].
3.  **Render Remaining Background:** Process the background octree $O_{bg}$ for the interval $[\tau_{curr}, \tau_{max}]$ using the accumulation function $F$. Update state $S \leftarrow F(O_{bg}, r, [\tau_{curr}, \tau_{max}], S)$ [S1, unofficial].
4.  **Final Composition:** Compute the final pixel color using the infinite-depth sky model:
    $$ C = C_{voxels} + T_{final} \cdot C_{sky} $$
    where $C_{voxels}$ is the aggregated color from all segments and $T_{final}$ is the final residual transmittance [S1, unofficial].

**Accumulation Function $F$:**
The function $F$ performs local, hardware-accelerated Morton-ordered voxel traversals within the specified bounded interval [S1, unofficial]. It filters the sorted voxel list of the target octree, isolating primitives within the active interval, and applies standard numerical integration, exponential-linear density activation, and alpha-compositing formulas [S1, unofficial]. It updates the state tuple $S$ by blending localized segment properties into the global state weighted by accumulated transmittance [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def algorithmic_multi_octree_render(O_bg, assets, transforms, ray_o, ray_d, intervals, epsilon_stop, tau_max, C_sky):
    """
    O_bg: Static background octree
    assets: List of dynamic asset octrees [O_1, ..., O_N]
    transforms: List of transformation matrices [T_t^1, ..., T_t^N]
    ray_o: Ray origin
    ray_d: Ray direction
    intervals: List of tuples [(tau_in_1, tau_out_1), ..., (tau_in_N, tau_out_N)]
    epsilon_stop: Transmittance threshold
    tau_max: Max ray distance
    C_sky: Sky color
    """
    # Initialize state tuple S = (C, D, N, T, tau_curr)
    C = 0.0
    D = 0.0
    N_vec = [0.0, 0.0, 0.0]
    T = 1.0
    tau_curr = 0.0
    
    N = len(assets)
    
    for i in range(N):
        tau_in = intervals[i][0]
        tau_out = intervals[i][1]
        
        # 1. Render background before object entry
        # F updates S based on O_bg, ray, interval, and current S
        C, D, N_vec, T = F_accumulate(O_bg, ray_o, ray_d, tau_curr, tau_in, C, D, N_vec, T)
        
        # 2. Transform ray to local canonical space
        # r_local = (T_t^i)^-1 * r
        # Assuming transforms[i] is the matrix T_t^i
        inv_transform = inverse_matrix(transforms[i])
        ray_o_local, ray_d_local = transform_ray(inv_transform, ray_o, ray_d)
        
        # 3. Render object in canonical local space
        C, D, N_vec, T = F_accumulate(assets[i], ray_o_local, ray_d_local, tau_in, tau_out, C, D, N_vec, T)
        
        # 4. Advance to next interval
        tau_curr = tau_out
        
        # 5. Early exit check
        if T < epsilon_stop:
            break
            
    # 6. Render remaining background
    C, D, N_vec, T = F_accumulate(O_bg, ray_o, ray_d, tau_curr, tau_max, C, D, N_vec, T)
    
    # 7. Final composition with sky
    C_final = C + T * C_sky
    
    return C_final, D, N_vec, T

def F_accumulate(octree, ray_o, ray_d, tau_start, tau_end, C_acc, D_acc, N_acc, T_acc):
    """
    Performs local, hardware-accelerated Morton-ordered voxel traversal
    within the bounded interval [tau_start, tau_end].
    Updates accumulated color, depth, normal, and transmittance.
    """
    # Filter voxels intersecting the ray segment within [tau_start, tau_end]
    # Sort voxels by Morton code (front-to-back)
    voxels = get_intersected_voxels(octree, ray_o, ray_d, tau_start, tau_end)
    voxels = sort_by_morton(voxels)
    
    for voxel in voxels:
        # Calculate density and color for the voxel segment
        # Using exponential-linear density activation and alpha-compositing
        density = get_density(voxel)
        color = get_color(voxel)
        normal = get_normal(voxel)
        
        # Compute alpha and transmittance for this step
        # Standard numerical integration over the voxel length
        alpha = compute_alpha(density, voxel.length)
        
        # Alpha compositing
        weight = alpha * T_acc
        C_acc = C_acc + weight * color
        D_acc = D_acc + weight * voxel.depth
        N_acc = N_acc + weight * normal
        T_acc = T_acc * (1.0 - alpha)
        
        # Early exit if transmittance is negligible
        if T_acc < 1e-6:
            break
            
    return C_acc, D_acc, N_acc, T_acc

def transform_ray(inv_transform, origin, direction):
    """
    Transforms ray origin and direction by inverse transformation matrix.
    """
    # Transform origin
    new_origin = transform_point(inv_transform, origin)
    # Transform direction (rotation only, assuming translation is handled by origin)
    new_direction = transform_vector(inv_transform, direction)
    return new_origin, new_direction

def inverse_matrix(M):
    """
    Computes the inverse of a 4x4 transformation matrix.
    """
    # Standard matrix inversion algorithm
    # Not given in the sources. [G]
    pass

def get_intersected_voxels(octree, ray_o, ray_d, tau_start, tau_end):
    """
    Returns list of voxels intersecting the ray segment.
    """
    # Traverse octree and collect voxels
    # Not given in the sources. [G]
    pass

def sort_by_morton(voxels):
    """
    Sorts voxels by their Morton code.
    """
    # Sort based on Morton code
    # Not given in the sources. [G]
    pass

def get_density(voxel):
    """
    Returns density of the voxel.
    """
    # Not given in the sources. [G]
    pass

def get_color(voxel):
    """
    Returns color of the voxel.
    """
    # Not given in the sources. [G]
    pass

def get_normal(voxel):
    """
    Returns normal of the voxel.
    """
    # Not given in the sources. [G]
    pass

def compute_alpha(density, length):
    """
    Computes alpha from density and length.
    """
    # Standard alpha computation
    # Not given in the sources. [G]
    pass
```

## Operating characteristics
*   **Time Complexity:** The method avoids the $O(N \log N)$ or higher cost of global spatial sorting for dynamic scenes by relying on $O(N)$ ray-box intersections and local Morton-ordered traversals within segments [S1, unofficial]. The cost is dominated by the number of dynamic assets $N$ and the voxel traversal within each segment [S1, unofficial].
*   **Memory Complexity:** Uses sparse voxel octrees for both background and dynamic assets, providing efficient memory usage compared to dense grids [S1, unofficial].
*   **Strengths:** Enables dynamic scenes with independently moving objects without costly runtime tree reconstruction; guarantees mathematically correct multi-volume occlusion and exact transmittance propagation across coordinate boundaries; reduces floating artifacts compared to complex runtime spatial tree sorting [S1, unofficial].
*   **Limitations:** Requires pre-computation of ray-box intersections and sorting of intervals; relies on the assumption that dynamic assets can be bounded by oriented 3D bounding boxes [S1, unofficial].
*   **When to use:** Ideal for dynamic scenes with multiple independently moving objects, such as autonomous driving simulations, where maintaining correct occlusion between dynamic entities and the static background is critical [S1, unofficial].
*   **When not to use:** Not suitable for scenes with highly complex, interpenetrating dynamic geometry that cannot be efficiently bounded by oriented 3D boxes, or where the overhead of interval sorting outweighs the benefits of dynamic updates [S1, unofficial].

## Background
Algorithmic Multi-Octree Render was introduced in the "DrivingVoxels" paper by T. Aguirre et al. [S1, unofficial]. It builds on the baseline SVRaster framework, which uses a tile-based rasterizer and Morton ordering for static scenes [S1, unofficial]. The method was developed to address the limitations of baseline frameworks in handling dynamic scenes, where independent moving objects change coordinates arbitrarily, making global Morton sorting inefficient [S1, unofficial]. It is used in the context of dynamic driving scene reconstruction and multi-modal supervision losses [S1, unofficial].

## Sources
[S1] DrivingVoxels: Compositional Sparse Voxel Rasterization for Dynamic Driving Scene Reconstruction, https://doi.org/10.48550/arxiv.2606.23031, fetched 2026-10-26 (unofficial)
