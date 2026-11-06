# Direct Multifield Volume Ray Casting of Fiber Surfaces

Also called: Direct Fiber Surface Rendering, Direct fiber surface ray casting, Direct ray casting of fiber surfaces, Fiber Surface Ray Casting, Fiber surface ray tracing

## Overview
Direct multifield volume ray casting of fiber surfaces is a rendering method for bivariate volume data that visualizes fiber surfaces (contours defined by a path in 2D range space) without explicit geometry extraction [S1]. It samples directly along rays in domain space and performs geometric tests in range space using a signed distance field derived from fiber surface control polygons (FSCPs) [S1]. Unlike mesh extraction methods like Marching Cubes, which produce piecewise-linear approximations and are costly for large data, this method enables real-time exploration, dynamic modification, and pixel-exact rendering with lower preprocess time [S1]. Compared to direct volume rendering with threshold-based transfer functions, it is roughly an order of magnitude faster for fine brush sizes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A bivariate volumetric field $\mathbf{f}: \mathbb{R}^3 \rightarrow \mathbb{R}^2$, defined by two scalar fields $f_1$ and $f_2$. A set of Fiber Surface Control Polygons (FSCPs) in the 2D range space $\mathbb{R}^2$. Camera parameters.
*   **Output:** A 2D image where pixels are colored based on the intersection of the viewing ray with the fiber surfaces defined by the FSCPs.

**Data Structures:**
1.  **Bivariate Volume Data:** Stored in 3D textures. Each voxel contains a pair $(f_1, f_2)$.
2.  **Distance Field:** A 2D texture of resolution $64 \times 64$ [S1]. Each channel stores the signed distance field for one FSCP. The value at coordinate $(u,v)$ is the minimum distance to the FSCP segments. Positive values indicate outside the polygon; negative values indicate inside [S1]. Up to 4 FSCPs can be stored simultaneously [S1].
3.  **Macrocell Grid:** A uniform 3D grid of macrocells. Each macrocell stores $n \times n \times n$ voxels (typically $n=3$, i.e., $3^3$ volume, though $4^3$ or $8^3$ are viable) [S1].
4.  **Boolean Grid:** A 3D texture storing boolean values for each macrocell. A value of `true` indicates the macrocell's range-space bounding box overlaps with the FSCP; `false` otherwise [S1]. Four boolean values are stored per macrocell to support up to 4 FSCPs [S1].

**Precomputation:**
1.  **Distance Field Computation:** For each FSCP, compute a 2D signed distance field. For each element in the $64 \times 64$ array, calculate the minimum distance to all segments of the FSCP. Assign positive values for outside points and negative values for inside points [S1]. This takes approximately 1-2 milliseconds for 4 FSCPs with 4 segments each [S1].
2.  **Macrocell Grid Construction:** Build a single-level uniform grid of macrocells. For each macrocell, compute the minimum and maximum values of both fields ($f_{min}, f_{max}, g_{min}, g_{max}$) over the $n \times n \times n$ voxels. To prevent missing fibers, include the minima and maxima of forward-adjacent voxels in the calculation [S1].
3.  **Boolean Grid Assignment:** For each macrocell, check if its range-space bounding box (defined by the min/max values) overlaps with the FSCP. If overlap exists, set the corresponding boolean value in the 3D texture to `true`; otherwise, set it to `false` [S1]. This step is parallelized using OpenMP, yielding a 2x-4x speedup with 6 threads [S1].

**Rendering Algorithm:**
The rendering is performed in a shader using a two-level loop structure [S1].

1.  **Outer Loop (Macrocell Traversal):** Traverse the macrocell grid using a 3D Digital Differential Analyzer (3DDDA) [S1].
2.  **Boolean Check:** For the current macrocell, fetch the precomputed boolean value from the 3D texture. If `false`, skip this macrocell [S1].
3.  **Inner Loop (Ray Marching):** If `true`, perform ray marching within the macrocell. The ray is divided into segments of size $\delta t$ (measured in voxel units). Typical segment sizes range from 0.00625 to 0.4, with 0.1 providing a balance between accuracy and performance [S1].
4.  **Data Fetch:** For each ray segment $[p_0, p_1]$, fetch the bivariate volume data at the endpoints:
    $$ \mathbf{fs}_0 = \text{fetch}(\mathbf{f}(p_0)) $$
    $$ \mathbf{fs}_1 = \text{fetch}(\mathbf{f}(p_1)) $$
5.  **Distance Fetch:** Fetch the signed distance field values at the range-space endpoints $\mathbf{fs}_0$ and $\mathbf{fs}_1$:
    $$ d_0 = \text{fetch}(\text{distance}(\mathbf{fs}_0)) $$
    $$ d_1 = \text{fetch}(\text{distance}(\mathbf{fs}_1)) $$
6.  **Intersection Test:** Apply the rule of signs (Bolzano's theorem). If $d_0 \cdot d_1 < 0$, a root (intersection) exists within the segment [S1].
7.  **Intersection Refinement:** Calculate the intersection point $\mathbf{fs}_t$ in range space using linear interpolation:
    $$ d_t = \left| \frac{d_0}{d_0 - d_1} \right| $$
    $$ \mathbf{fs}_t = \mathbf{fs}_0 + d_t \cdot (\mathbf{fs}_1 - \mathbf{fs}_0) $$
    The corresponding domain space position $p_t$ is interpolated similarly [S1].
8.  **Shading:** Compute the gradient of the implicit fiber surface at $p_t$. The fiber surface is defined by $g(\mathbf{x}) = a f_1(\mathbf{x}) + b f_2(\mathbf{x})$, where $(a,b)$ is the coordinate of $\mathbf{fs}_t$ in the range space. The gradient is:
    $$ \nabla C = a \nabla f_1(p_t) + b \nabla f_2(p_t) $$
    Compute $\nabla f_1$ and $\nabla f_2$ using multiple central-difference neighbor stencils [S1]. Shade the point using standard lighting models (e.g., Phong) with this normal [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Precomputation (CPU)
def precompute_fiber_surfaces(volume_data, fscps, macrocell_size=3):
    # 1. Compute Distance Fields
    distance_fields = []
    for fscp in fscps:
        # Create 64x64 2D texture
        df = zeros((64, 64))
        for u in range(64):
            for v in range(64):
                point = (u, v)
                # Calculate min distance to FSCP segments
                dist = min_distance_to_polygon(point, fscp)
                # Sign: positive outside, negative inside
                if is_inside_polygon(point, fscp):
                    df[u, v] = -dist
                else:
                    df[u, v] = dist
        distance_fields.append(df)
    
    # 2. Build Macrocell Grid and Boolean Grid
    num_macrocells_x = volume_dims[0] // macrocell_size
    num_macrocells_y = volume_dims[1] // macrocell_size
    num_macrocells_z = volume_dims[2] // macrocell_size
    boolean_grid = zeros((num_macrocells_x, num_macrocells_y, num_macrocells_z, 4))
    
    for x in range(num_macrocells_x):
        for y in range(num_macrocells_y):
            for z in range(num_macrocells_z):
                # Get voxel range for this macrocell
                # Include forward-adjacent voxels to prevent missing fibers
                min_vals, max_vals = get_voxel_min_max(volume_data, x, y, z, macrocell_size)
                
                # Check overlap with each FSCP
                for i, fscp in enumerate(fscps):
                    # Check if range box [min_vals, max_vals] overlaps fscp
                    if boxes_overlap(min_vals, max_vals, fscp):
                        boolean_grid[x, y, z, i] = 1.0
                    else:
                        boolean_grid[x, y, z, i] = 0.0
    return distance_fields, boolean_grid

# Rendering (GPU Shader)
def render_fiber_surface(ray_origin, ray_direction, distance_fields, boolean_grid, macrocell_size):
    # 3DDDA Traversal
    macrocell = traverse_3d_dda(ray_origin, ray_direction, boolean_grid)
    
    while macrocell is not None:
        # Check if fiber surface exists in this macrocell
        # boolean_grid stores 4 booleans for up to 4 FSCPs
        has_fiber = any(boolean_grid[macrocell.x, macrocell.y, macrocell.z, :])
        
        if has_fiber:
            # Ray marching within macrocell
            segment_start = macrocell.entry_point
            segment_end = macrocell.exit_point
            
            # Determine segment size delta_t (e.g., 0.1 voxel units)
            delta_t = 0.1 
            
            t = 0.0
            while t < 1.0:
                p0 = ray_origin + ray_direction * (segment_start + t * (segment_end - segment_start))
                p1 = ray_origin + ray_direction * (segment_start + (t + delta_t) * (segment_end - segment_start))
                
                # Fetch bivariate data
                fs0 = fetch_bivariate_volume(p0)
                fs1 = fetch_bivariate_volume(p1)
                
                # Fetch distance field values
                # fs0 and fs1 are in range space (u,v)
                d0 = fetch_distance_field(fs0, distance_fields)
                d1 = fetch_distance_field(fs1, distance_fields)
                
                # Rule of signs test
                if d0 * d1 < 0:
                    # Intersection found
                    dt = abs(d0 / (d0 - d1))
                    fs_t = fs0 + dt * (fs1 - fs0)
                    
                    # Interpolate domain position
                    p_t = p0 + dt * (p1 - p0)
                    
                    # Compute gradient
                    # (a, b) are coordinates of fs_t in range space
                    a, b = fs_t.u, fs_t.v
                    grad_f1 = compute_gradient_volume(p_t, field_index=0)
                    grad_f2 = compute_gradient_volume(p_t, field_index=1)
                    normal = a * grad_f1 + b * grad_f2
                    
                    # Shade
                    color = apply_lighting(p_t, normal, ray_direction)
                    return color
                
                t += delta_t
        
        macrocell = traverse_3d_dda_next(macrocell)
        
    return background_color
```

## Operating characteristics
*   **Time Complexity:** The worst-case time complexity for uniform grid rendering is $O(N^{3/2})$, where $N$ is the number of voxels [S1]. The best-case is $O(1)$ due to empty space skipping [S1]. Histogram computation is $O(N)$ [S1].
*   **Memory Complexity:** Requires storage for the bivariate volume data, the $64 \times 64$ distance fields (2D textures), and the boolean grid (3D texture) [S1].
*   **Performance:** On an NVIDIA GeForce GT 650M mobile GPU at 1024x1024 resolution, the method achieves 16 fps for a coal combustion dataset [S1]. For the Homo-Lumo dataset ($4^3$ macrocells, segment size 0.1), it achieves >20 fps [S1]. On an NVIDIA GeForce GTX 780, the 512^3 Enzo dataset runs at 2.76 fps, which is 0.77x the speed of the 256^3 dataset (3.98 fps) despite having 8x more voxels [S1].
*   **Parameters:**
    *   **Macrocell Size:** Typically $3^3$ voxels provides the best performance balance, though $4^3$ or $8^3$ are viable for larger datasets [S1]. Smaller sizes increase performance but require more traversal steps [S1].
    *   **Segment Size:** Smaller sizes improve accuracy but reduce performance. Typical values are 0.1 voxel units [S1].
    *   **Distance Field Resolution:** $64 \times 64$ is sufficient [S1].
*   **Strengths:** Real-time interactive exploration, pixel-exact rendering, support for higher-order interpolation (e.g., tricubic B-splines) without intractable analytical intersections, and lower preprocess time compared to mesh extraction [S1].
*   **Limitations:** Sensitive to segment size and FSCP frequency. Less robust than analytical methods; prone to false negatives (missed intersections) and false positives (spurious intersections) if sampling is insufficient [S1]. Cannot guarantee root isolation [S1]. Distance fields for multiple FSCPs cannot be merged [S1].

## Background
*   **Origin:** The method was introduced in a paper by [S1] (IEEE Transactions on Visualization and Computer Graphics, 2016). It addresses the challenge of visualizing multifield data, where reducing data to comprehensible geometry is difficult.
*   **History:** Fiber surfaces generalize isosurfaces to bivariate data by taking the inverse image of a 1-manifold path in range space [S1]. Previous methods relied on mesh extraction (e.g., Marching Cubes), which is non-interactive for large data [S1]. This method enables direct ray casting without explicit geometry extraction [S1].
*   **Usage:** Implemented in the Uintah framework for analyzing fluid dynamic simulations, specifically coal combustion systems (BSF and OFC datasets) [S1]. Also used in cosmology and chemistry simulations [S1].

## Sources
[S1] Direct Multifield Volume Ray Casting of Fiber Surfaces, https://doi.org/10.1109/tvcg.2016.2599040, fetched 2026-10-26
