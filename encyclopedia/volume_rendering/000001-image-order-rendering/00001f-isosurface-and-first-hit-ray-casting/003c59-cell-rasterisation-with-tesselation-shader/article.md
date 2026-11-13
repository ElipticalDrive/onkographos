# Cell Rasterisation with Tesselation Shader

Also called: Billboard Splatting with Tesselation, Clustered Grid Cell Rasterisation

## Overview
Cell Rasterisation with Tesselation Shader is an image-order rendering technique for isosurfaces that replaces traditional vertex shaders with tessellation shaders to efficiently rasterize clusters of grid cells as point sprites. It solves the performance bottleneck of rendering large volumes by grouping active cells into clusters, expanding them into individual points in the tessellation stage, and performing ray-box intersection and volume sampling in the fragment shader. Unlike its sibling [Volumetric Isosurface Rendering with Intersection Refinement](958-volumetric-isosurface-rendering-with-intersection-refinement.md) which relies on precise ray marching, this method leverages hardware rasterization of point primitives to determine surface intersections, offering a balance between the speed of billboard splatting and the accuracy of raycasting.

## How it works
**Inputs and Outputs:**
*   **Input:** A Clustered Grid Cell (CGC) data structure containing metacells and grid cells. Each cluster is represented as a single `GL_PATCHES` primitive. The volume scalar field data is stored in GPU memory.
*   **Output:** A color buffer with the rendered isosurface and a G-buffer containing geometric attributes (normals, positions) for deferred shading.

**Data Structures:**
*   **Cluster Patch:** A single `GL_PATCHES` primitive per cluster. The tessellation shader expands this into points for each cell in the cluster.
*   **Metacell/Cell Metadata:**
    *   Grid position: 16 bits per component (x, y, z). This limits the maximum representable grid size to $128K \times 128K \times 128K$ voxels [S1].
    *   Corner values: Two RGBA values per cell are stored to allow interpolation of scalar values in the fragment shader [S1].
    *   Cluster Bound: The last metacell of each cluster stores the cluster bounding box instead of cell metadata [S1].
*   **Visibility Buffer:** An `RG32UI` texture storing a bit per cluster to indicate visibility status across frames [S1].

**Algorithm Steps:**

1.  **Tessellation (Cluster Expansion):**
    The tessellation shader receives one `GL_PATCHES` primitive per cluster. It iterates through the cells defined in the cluster's metadata. For each active cell, it generates a point primitive.
    *   **Cell Normal Computation:** The tessellation shader computes a normal vector for the cell using the stored corner scalar values. This normal is used for back-face culling [S1].
    *   **Culling:** If the cell is back-facing relative to the camera, it is discarded [S1].

2.  **Rasterization (Billboard Splatting):**
    The generated points are rasterized as billboards (point sprites). The size of the billboard corresponds to the cell's spatial extent.
    *   **Reduced Rate Rendering:** Clusters that were culled in the previous frame are rendered at a reduced rate (1/8th) with at most one point per metacell per frame until they become visible again [S1].

3.  **Fragment Shader (Ray-Box Intersection):**
    For each rasterized fragment of the billboard:
    *   **Ray Definition:** Define the view ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ originating from the camera.
    *   **Ray-Box Intersection:** Perform a ray-box intersection test between the view ray and the cell's bounding box. The implementation uses the efficient slab test described by Majercik et al. [S1]. If no intersection occurs, the fragment is discarded.
    *   **Intersection Point:** Calculate the entry and exit points of the ray within the cell's bounding box.

4.  **Volume Sampling and Isosurface Detection:**
    *   **Interpolation:** Sample the scalar field $f(\mathbf{p})$ along the ray segment inside the cell. The scalar values are obtained by manually performing tri-linear interpolation using the stored corner values in the fragment shader [S1].
    *   **Sample Count:** The method uses a fixed number of samples per ray. The sources specify that **3 samples** per ray are used, which was found sufficient for accurate intersection detection [S1].
    *   **Intersection Check:** Detect the isosurface intersection where $f(\mathbf{p}) = \tau$ (threshold). This is typically identified by a sign change or threshold crossing between consecutive samples [G].

5.  **Normal Computation:**
    *   Compute the gradient $\nabla f$ at the intersection point using the tri-linearly interpolated scalar values. This gradient serves as the surface normal for shading [S1].

6.  **G-Buffer Writing:**
    *   Rasterize the cell's geometric attributes (position, normal) to a Geometry Buffer (G-buffer) [S1].

7.  **Deferred Shading:**
    *   In a separate pass, compute normalized Blinn-Phong shading using the data from the G-buffer. This includes ambient, diffuse, and specular components [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Tessellation Shader
def tessellation_shader(cluster_patch):
    # cluster_patch contains metadata for all cells in the cluster
    for cell in cluster_patch.cells:
        # Compute cell normal from stored corner values
        normal = compute_cell_normal(cell.corner_values)
        
        # Back-face culling
        if dot(normal, view_direction) > 0:
            discard()
        
        # Emit point for the cell
        emit_point(cell.position, cell.size, normal)

# Fragment Shader
def fragment_shader(point_sprite):
    # 1. Ray-Box Intersection
    # Ray is defined by camera origin and direction
    hit = ray_box_intersection_slab_test(ray, point_sprite.bounding_box)
    
    if not hit.intersects:
        discard()
    
    t_start, t_end = hit.entry_t, hit.exit_t
    
    # 2. Volume Sampling (Fixed 3 samples)
    num_samples = 3
    samples = []
    for i in range(num_samples):
        t = lerp(t_start, t_end, i / (num_samples - 1))
        p = ray.origin + t * ray.direction
        
        # Manual tri-linear interpolation of scalar value from corner values
        scalar = tri_linear_interpolate(p, point_sprite.corner_values)
        samples.append((p, scalar))
    
    # 3. Isosurface Detection
    intersection_found = False
    intersection_point = None
    intersection_normal = None
    
    for i in range(len(samples) - 1):
        p1, s1 = samples[i]
        p2, s2 = samples[i+1]
        
        # Check for threshold crossing
        if (s1 - threshold) * (s2 - threshold) < 0:
            # Linear interpolation to refine intersection point
            t_refine = (threshold - s1) / (s2 - s1)
            intersection_point = lerp(p1, p2, t_refine)
            
            # Compute normal from interpolated samples
            intersection_normal = compute_gradient(intersection_point, point_sprite.corner_values)
            intersection_found = True
            break
    
    if not intersection_found:
        discard()
    
    # 4. Write to G-Buffer
    write_gbuffer(intersection_point, intersection_normal)
```

**Parameters:**
*   **Samples per ray:** 3 [S1].
*   **Grid Position Storage:** 16 bits per component [S1].
*   **Culling Rate:** 1/8 for previously culled clusters [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The 16-bit storage for grid positions limits the maximum grid resolution to $128K^3$ [S1].
*   **Culled Clusters:** To avoid flickering, culled clusters are rendered at a reduced rate rather than being completely skipped [S1].
*   **Sample Sufficiency:** Only 3 samples are used; this may lead to aliasing or missed intersections in highly detailed regions, though the source claims it is sufficient for their use case [S1].

## Subtopics

### Cell Rasterisation via Billboard Splatting
A variant of the method that uses a vertex shader instead of a tessellation shader to render active grid cells as `GL_POINTS` primitives. It performs ray-box intersection and interpolates samples along the ray using stored corner values (two RGBA values per cell) to find intersections and compute normals. It uses 3 samples per ray [S1]. This variant is simpler but less efficient for large clusters compared to the tessellation-based approach which expands clusters in the tessellation stage [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of active cells and clusters. The tessellation shader expands clusters, and the fragment shader performs a fixed number of samples (3) per ray. The use of a visibility buffer reduces redundant rasterization of occluded clusters [S1].
*   **Memory Complexity:** Requires storage for the CGC data structure, including 16-bit grid positions and corner values for each cell. The visibility buffer adds minimal overhead [S1].
*   **Performance:** Fast due to hardware rasterization of point sprites and deferred shading. The tessellation shader approach is more efficient than vertex shader-based billboard splatting for large clusters [S1].
*   **Strengths:** High performance for large volumes; leverages GPU hardware rasterization; deferred shading allows for efficient lighting calculations [S1].
*   **Limitations:** Limited to 16-bit grid positions; fixed sample count may affect accuracy; requires a visibility buffer for optimal performance [S1].
*   **When to use:** Real-time isosurface rendering of large volumes where performance is critical and moderate accuracy is acceptable.
*   **When not to use:** When high precision is required (e.g., scientific visualization requiring sub-voxel accuracy) or when grid sizes exceed $128K^3$.

## Background
Cell Rasterisation was proposed by Liu et al. [13] as a fast method for rasterizing active grid cells of a volume. The technique was originally implemented using vertex shaders for billboard splatting. The variant using tessellation shaders and the Clustered Grid Cell (CGC) data structure was developed to improve performance by reducing the number of draw calls and leveraging the tessellation pipeline for cluster expansion. The method is part of the broader family of image-order rendering techniques for isosurfaces.

## Sources
[S1] Clustered Grid Cell Data Structure for Isosurface Rendering, https://doi.org/10.24132/jwscg.2020.28.2, fetched 2026-10-26
