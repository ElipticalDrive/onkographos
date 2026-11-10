# Direct Multifield Volume Ray Casting of Fiber Surfaces

Also called: Direct Fiber Surface Rendering, Direct fiber surface ray casting, Direct ray casting of fiber surfaces, Fiber Surface Ray Casting, Fiber surface ray tracing

## Overview
Direct multifield volume ray casting of fiber surfaces is a rendering method for bivariate volume data that visualizes fiber surfaces (contours defined by a path in 2D range space) without explicit geometry extraction [S1]. It samples directly along rays in domain space and performs geometric tests in range space using a signed distance field derived from fiber surface control polygons (FSCPs) [S1]. Unlike mesh extraction methods such as Marching Cubes, which produce piecewise-linear approximations and are costly for large data, this method enables real-time exploration, dynamic modification, and pixel-exact rendering with lower preprocess time [S1]. Compared to direct volume rendering with threshold-based transfer functions, it is roughly an order of magnitude faster for fine brush sizes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A bivariate volumetric field $\mathbf{f}: \mathbb{R}^3 \rightarrow \mathbb{R}^2$, defined by two scalar fields $f_1(\mathbf{x})$ and $f_2(\mathbf{x})$. A Fiber Surface Control Polygon (FSCP), which is a closed polygon in the 2D range space $\mathbb{R}^2$ defining the target fiber surface.
*   **Output:** A rendered image where pixels corresponding to the fiber surface are shaded, and others are transparent or background-colored.

**Data Structures:**
1.  **Bivariate Volume Data:** Stored in 3D textures. Each voxel contains a pair of values $(f_1, f_2)$.
2.  **Distance Field:** A 2D texture of resolution $R \times R$ (typically $64 \times 64$). Each channel stores the signed distance from a point in range space to the nearest segment of the FSCP. Positive values indicate outside the polygon; negative values indicate inside [S1]. Up to 4 FSCPs can be visualized simultaneously using up to 4 channels [S1].
3.  **Macrocell Grid:** A uniform 3D grid of macrocells. Each macrocell covers an $n \times n \times n$ volume of voxels (typically $n=3$, i.e., $2^3$ or $3^3$ voxels, though $4^3$ or $8^3$ are used for larger datasets) [S1].
4.  **Boolean Grid:** A 3D texture storing boolean values for each macrocell. A value of `true` indicates the macrocell's range-space bounding box overlaps with the FSCP; `false` indicates no overlap [S1]. Four boolean values are stored per macrocell to support up to 4 FSCPs [S1].

**Precomputation:**
1.  **Distance Field Computation:** For each FSCP, compute a 2D signed distance field. For each texel in the $R \times R$ grid, calculate the minimum distance to all segments of the FSCP. Assign positive values for points outside and negative for points inside [S1]. This takes approximately 1-2 milliseconds for 4 FSCPs with 4 segments each [S1].
2.  **Macrocell Grid Construction:** Build a single-level uniform grid. For each macrocell, compute the minimum and maximum values of both fields ($f_{min}, f_{max}, g_{min}, g_{max}$) over the voxels it contains. To prevent missing fibers, include the minima and maxima of forward-adjacent voxels [S1].
3.  **Boolean Grid Assignment:** For each macrocell, check if its range-space bounding box (defined by min/max values) overlaps with the FSCP. If overlap exists, set the corresponding boolean value in the 3D texture to `true`; otherwise, set it to `false` [S1]. This step is parallelized using OpenMP, yielding a 2x-4x speedup with 6 threads [S1].

**Rendering Algorithm:**
The rendering is performed in a shader using a two-level loop structure: outer loop for macrocell traversal and inner loop for ray marching within the macrocell.

1.  **Macrocell Traversal:** Traverse the macrocell grid using a 3D Digital Differential Analyzer (3DDDA) [S1].
2.  **Boolean Check:** For the current macrocell, fetch the precomputed boolean value from the 3D texture. If `false`, skip the macrocell (empty space optimization) [S1]. If `true`, proceed to ray marching [S1].
3.  **Ray Marching:** Within the macrocell, march along the ray in segments of size $\delta t$ (measured in voxel units). Typical segment sizes are 0.1, but smaller sizes (e.g., 0.025) improve accuracy at the cost of performance [S1].
4.  **Data Fetching:** For each ray segment $[p_0, p_1]$, fetch the bivariate volume data at the endpoints:
    $$ \mathbf{fs}_0 = \text{fetch}(\mathbf{f}(p_0)) = (f_1(p_0), f_2(p_0)) $$
    $$ \mathbf{fs}_1 = \text{fetch}(\mathbf{f}(p_1)) = (f_1(p_1), f_2(p_1)) $$
5.  **Distance Fetching:** Fetch the signed distance field values at the range-space endpoints:
    $$ d_0 = \text{fetch}(\text{distance}(\mathbf{fs}_0)) $$
    $$ d_1 = \text{fetch}(\text{distance}(\mathbf{fs}_1)) $$
6.  **Intersection Test:** Apply the rule of signs (Bolzano's theorem). If $d_0 \cdot d_1 < 0$, a root (intersection with the fiber surface) exists within the segment [S1].
7.  **Intersection Refinement:** Calculate the intersection point $\mathbf{fs}_t$ in range space using linear interpolation:
    $$ d_t = \left| \frac{d_0}{d_0 - d_1} \right| $$
    $$ \mathbf{fs}_t = \mathbf{fs}_0 + d_t \cdot (\mathbf{fs}_1 - \mathbf{fs}_0) $$
    Map $\mathbf{fs}_t$ back to the domain space parameter $t$ to find the 3D position $p_t$ [S1].
8.  **Shading:** Compute the gradient of the implicit fiber surface at $p_t$. The fiber surface is defined by the linear combination $g(\mathbf{x}) = a f_1(\mathbf{x}) + b f_2(\mathbf{x})$, where $(a, b)$ is the coordinate of $\mathbf{fs}_t$ in the range space. The gradient is:
    $$ \nabla C = a \nabla f_1(p_t) + b \nabla f_2(p_t) $$
    Compute $\nabla f_1$ and $\nabla f_2$ using multiple central-difference neighbor stencils to capture higher frequency variations [S1]. Use $\nabla C$ for Phong-like shading (ambient, diffuse, specular) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Precomputation (CPU/OpenMP)
def precompute_fiber_surface(volume_data, fscp, macrocell_size_n):
    # 1. Compute 2D Distance Field
    # Resolution R=64
    distance_field = zeros((R, R))
    for y in range(R):
        for x in range(R):
            # Map grid coordinates to range space coordinates (f, g)
            f_val, g_val = grid_to_range(x, y)
            # Compute signed distance to FSCP segments
            distance_field[y, x] = signed_distance_to_fscp(f_val, g_val, fscp)
    return distance_field

def build_macrocell_grid(volume_data, distance_field, macrocell_size_n):
    # 2. Build Macrocell Grid and Boolean Grid
    num_macrocells_x = volume_dims[0] // macrocell_size_n
    num_macrocells_y = volume_dims[1] // macrocell_size_n
    num_macrocells_z = volume_dims[2] // macrocell_size_n
    boolean_grid = zeros((num_macrocells_x, num_macrocells_y, num_macrocells_z))
    
    for z in range(num_macrocells_z):
        for y in range(num_macrocells_y):
            for x in range(num_macrocells_x):
                # Get voxel bounds for this macrocell
                # Include forward-adjacent voxels to prevent missing fibers
                v_min, v_max = get_macrocell_voxel_bounds(x, y, z, macrocell_size_n)
                
                # Compute min/max of both fields in this volume
                f_min, f_max, g_min, g_max = compute_field_extents(volume_data, v_min, v_max)
                
                # Check overlap with FSCP in range space
                # The range box is (f_min, g_min) to (f_max, g_max)
                if box_overlaps_fscp((f_min, g_min, f_max, g_max), fscp):
                    boolean_grid[z, y, x] = 1
                else:
                    boolean_grid[z, y, x] = 0
    return boolean_grid

# Rendering (GPU Shader)
def render_fiber_surface(ray_origin, ray_direction, volume_texture, distance_field_texture, boolean_grid_texture, macrocell_size_n, segment_size):
    # 3DDDA Traversal
    macrocell_pos = get_initial_macrocell(ray_origin)
    
    while macrocell_pos is within_volume:
        # Fetch boolean value
        is_active = boolean_grid_texture.sample(macrocell_pos)
        
        if is_active:
            # Ray marching within macrocell
            t_start, t_end = get_macrocell_ray_bounds(ray_origin, ray_direction, macrocell_pos)
            t = t_start
            
            while t < t_end:
                t_next = min(t + segment_size, t_end)
                
                # Fetch bivariate data at segment endpoints
                p0 = ray_origin + t * ray_direction
                p1 = ray_origin + t_next * ray_direction
                
                fs0 = volume_texture.sample(p0) # Returns (f1, f2)
                fs1 = volume_texture.sample(p1) # Returns (f1, f2)
                
                # Fetch distance field values
                d0 = distance_field_texture.sample(fs0)
                d1 = distance_field_texture.sample(fs1)
                
                # Rule of signs test
                if d0 * d1 < 0:
                    # Intersection found
                    dt = abs(d0 / (d0 - d1))
                    fs_t = fs0 + dt * (fs1 - fs0)
                    
                    # Map back to domain t
                    t_hit = t + dt * (t_next - t)
                    p_hit = ray_origin + t_hit * ray_direction
                    
                    # Shading
                    # Gradient calculation
                    a, b = fs_t # Coordinates in range space
                    grad_f1 = compute_gradient(volume_texture, p_hit, field_index=0)
                    grad_f2 = compute_gradient(volume_texture, p_hit, field_index=1)
                    normal = a * grad_f1 + b * grad_f2
                    
                    color = phong_shading(p_hit, normal, ray_direction)
                    return color
                
                t = t_next
        
        # Advance to next macrocell
        macrocell_pos = next_macrocell(macrocell_pos, ray_direction)
        
    return background_color
```

**Parameters:**
*   **Macrocell size ($n$):** Number of voxels per dimension in a macrocell. Typical value is 3 ($2^3$ or $3^3$ volume), though $4^3$ or $8^3$ are used for larger datasets. Smaller sizes increase performance but require more traversal steps [S1].
*   **Segment size:** Length of the ray marching step in voxel units. Smaller sizes improve accuracy but reduce performance. Typical values are 0.1, with 0.025 or 0.00625 for higher accuracy [S1].
*   **Distance field resolution:** $64 \times 64$ array is used [S1].
*   **FSCP:** Closed polygon in range space. Can be convex or concave [S1].

**Edge Cases and Pitfalls:**
*   **False Negatives:** If segment endpoints have distance field values with the same sign, the method may miss the fiber surface intersection [S1].
*   **False Positives:** If segment endpoints have distance field values with different signs, false intersections may be reported [S1].
*   **Segment Size Sensitivity:** If the ray segment size is too large, the image in range space becomes a complex curve intersecting fiber segments multiple times. Sufficient sampling is required to approximate the path as piecewise-linear chords [S1].
*   **Floating Point Error:** For very small segment sizes, accuracy is limited by floating point precision and the discretization of the distance field [S1].
*   **Missing Surface Parts:** If the FSCP is very small (e.g., a thin line), the intersection detection may lose parts of the fiber surface [S1].
*   **Concave Polygons:** Cross product alone cannot determine inside/outside for concave polygons. The distance field handles this by using signed distances (positive outside, negative inside) [S1].
*   **Multiple FSCPs:** Distance fields for multiple FSCPs cannot be merged. Each FSCP requires its own distance field [S1].
*   **Exact Extraction:** For trilinearly interpolated cubic cells, exact fiber surfaces cannot be extracted as a polygonal mesh because individual fibers may have multiple connected components and intersections can be complex (e.g., at least 12 fiber components in specific cases) [S1].

## Operating characteristics
*   **Time Complexity:** Worst-case time complexity for uniform grid rendering is $O(N^{3/2})$, where $N$ is the number of voxels [S1]. Best-case is $O(1)$ due to empty space skipping [S1]. Histogram computation is $O(N)$ [S1].
*   **Memory Complexity:** Requires storage for the bivariate volume data, the 2D distance field ($64 \times 64$ per FSCP), and the 3D boolean grid for macrocells [S1].
*   **Performance:**
    *   On an NVIDIA GeForce GT 650M mobile GPU, the method achieves 16 fps at 1024x1024 resolution for a coal combustion simulation dataset [S1].
    *   For the Homo-Lumo $4^3$ dataset, performance is >20 fps [S1].
    *   For the 256^3 Enzo dataset, performance is 3.98 fps on a mobile GPU. The 512^3 dataset runs at 2.76 fps, which is 0.77x the speed of the 256^3 dataset despite having 8x more voxels [S1].
    *   Using a 3D tricubic B-spline filter increases rendering time by 3x compared to the base method (8 fps vs 23 fps on Homo-Lumo) [S1].
    *   Preprocess speedup: 6 OpenMP threads yield a 2x-4x speedup depending on the dataset [S1].
*   **Strengths:** Real-time exploration, dynamic modification of FSCPs, pixel-exact rendering, support for higher-order interpolation (e.g., B-splines) without intractable analytical intersection calculations, and efficient empty space skipping via macrocells [S1].
*   **Limitations:** Less robust than analytical intersection methods; sensitive to segment size and FSCP frequency; cannot guarantee root isolation; distance fields for multiple FSCPs cannot be merged [S1].
*   **When to use:** When interactive exploration of bivariate fiber surfaces is required, especially for large datasets where mesh extraction is too slow, and when higher-order interpolation is desired [S1].
*   **When not to use:** When exact geometric extraction of the fiber surface mesh is required, or when the data has very high frequency relative to the segment size, leading to excessive false positives/negatives [S1].

## Background
*   **Origin:** The method was introduced in the paper "Direct Multifield Volume Ray Casting of Fiber Surfaces" [S1].
*   **History:** Fiber surfaces generalize isosurfaces to bivariate data by taking the inverse image of a 1-manifold path in the range space [S1]. Previous methods relied on mesh extraction (e.g., Marching Cubes), which is costly and non-interactive [S1]. This method was developed to enable real-time exploration of multifield data, such as coal combustion simulations in the Uintah framework [S1].
*   **Usage:** Used in the Uintah framework for analyzing fluid dynamic simulations, specifically coal combustion systems (BSF and OFC datasets) [S1]. Implemented in a FSCP editor allowing interactive exploration [S1].

## Sources
[S1] Direct Multifield Volume Ray Casting of Fiber Surfaces, https://doi.org/10.1109/tvcg.2016.2599040, fetched 2026-10-26
