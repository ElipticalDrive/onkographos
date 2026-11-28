# Boundary Cell-Based Ray Casting

Also called: BC-RC, Cell-Based Ray Casting, Improved PARC

## Overview
Boundary Cell-Based Ray Casting (BC-RC) is a fast serial ray casting algorithm that accelerates volume rendering by skipping empty space using boundary cells [S1, S2, unofficial]. It improves upon the PARC algorithm by obtaining a more accurate object boundary estimation with less computational overhead using incremental parallel projection for each boundary cell [S1, S2, unofficial]. Unlike PARC, which uses pre-calculated sub-volumes whose projection is expensive when small, BC-RC uses tiny grid cells for more accurate boundary estimation and employs fast incremental parallel projection with templates to reduce projection overhead [S2, unofficial].

## How it works
The algorithm consists of three main steps: (1) detect boundary cells in a preprocessing stage; (2) project the boundary cells onto the image plane to produce intersection distance values for each pixel; and (3) for each viable pixel that intersects the object, sample and composite starting from the intersection [S1, S2, unofficial].

### Inputs and Outputs
**Inputs:**
*   Volumetric data set: A regular grid of volume elements (voxels) with scalar values [S1, S2, unofficial].
*   Camera parameters: View direction and image plane dimensions for parallel projection [S1, S2, unofficial].

**Outputs:**
*   Rendered image: Computed via ray traversal from closest to farthest intersection [S1, S2, unofficial].
*   Z-buffers: Front ($Z_n$) and Rear ($Z_r$) buffers storing intersection distances [S2, unofficial].

### Data Structures
1.  **Boundary Cell Flag Array ($F$):**
    *   Layout: A 3D array indexed by cell coordinates $(i, j, k)$.
    *   Content: Binary flag $F(i, j, k)$. Value 1 indicates the cell contains the object boundary; 0 indicates empty or fully inside/outside (not a boundary cell) [S1, S2, unofficial].
2.  **Front Z-Buffer ($Z_n$):**
    *   Layout: 2D array indexed by image pixel coordinates.
    *   Content: Stores the closest intersection distance to the object boundary for each pixel [S2, unofficial].
3.  **Rear Z-Buffer ($Z_r$):**
    *   Layout: 2D array indexed by image pixel coordinates.
    *   Content: Stores the farthest intersection distance to the object boundary for each pixel [S2, unofficial].

### Definitions
*   **Grid Cell:** The volume contained within the rectangular box bounded by eight corner voxels from two adjacent parallel slices [S1, S2, unofficial].
*   **Boundary Cell:** A grid cell containing the object boundary inside the volume [S1, S2, unofficial].
*   **Parallel Projection:** A projection method where the projected area of every cell has the same shape and size in the image plane. Only the projected position and the distance of the cell center to the image plane differ from cell to cell [S1, S2, unofficial].

### Steps

1.  **Boundary Cell Detection (Offline Preprocessing):**
    *   Scan the volume data [S1, S2, unofficial].
    *   For each grid cell $C(i, j, k)$, determine if it contains the object boundary [S1, S2, unofficial].
    *   If a boundary cell is found, set its flag $F(i, j, k) = 1$ [S1, S2, unofficial].
    *   If not, set $F(i, j, k) = 0$ [S2, unofficial].
    *   *Note:* This information is viewpoint-independent [S1, S2, unofficial].

2.  **Boundary Cell Projection (Online):**
    *   Initialize front z-buffer $Z_n$ and rear z-buffer $Z_r$ for all pixels to infinity/empty [G].
    *   Project boundary cells onto the image plane using parallel projection [S1, S2, unofficial].
    *   Only cells with $F(i, j, k) = 1$ are projected [S1, S2, unofficial].
    *   **Incremental Projection:**
        *   Project the first cell $C(0, 0, 0)$ onto the image plane to obtain its initial projected position and depth [S1, S2, unofficial].
        *   Calculate the projected positions and depths of remaining cells using incremental vectors with only addition operations [S1, S2, unofficial].
        *   The projected coordinates $(u, v)$ and depth $d$ for cell $(i, j, k)$ are computed as:
            $$ u = u_0 + i \cdot \Delta u_i + j \cdot \Delta u_j + k \cdot \Delta u_k $$
            $$ v = v_0 + i \cdot \Delta v_i + j \cdot \Delta v_j + k \cdot \Delta v_k $$
            $$ d = d_0 + i \cdot \Delta d_i + j \cdot \Delta d_j + k \cdot \Delta d_k $$
        *   *Note:* This avoids expensive per-cell projection calculations [S2, unofficial].
    *   **Z-Buffer Update:**
        *   For each projected boundary cell, determine the intersection distances along the rays cast from the image pixels [S2, unofficial].
        *   Update $Z_n$ with the closest intersection distance [S2, unofficial].
        *   Update $Z_r$ with the farthest intersection distance [S2, unofficial].

3.  **Ray Traversal:**
    *   Iterate over each pixel in the image plane [S1, S2, unofficial].
    *   Check if the pixel is viable: if $Z_n$ and $Z_r$ indicate an intersection (i.e., not empty) [S1, S2, unofficial].
    *   For viable pixels, cast a ray [S1, S2, unofficial].
    *   Traverse the ray from the closest intersection distance ($Z_n$) to the farthest intersection distance ($Z_r$) [S1, S2, unofficial].
    *   Sample and composite the volume data along the ray [S1, S2, unofficial].
    *   Incorporate optimizations such as adaptive image sampling and early ray termination [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def boundary_cell_based_ray_casting(volume_data, image_width, image_height, cell_size, view_direction):
    """
    Boundary Cell-Based Ray Casting algorithm.
    volume_data: 3D list of scalars.
    image_width, image_height: int.
    cell_size: float.
    view_direction: (dx, dy, dz) normalized vector.
    Returns: 2D list of colors.
    """
    # Step 1: Boundary Cell Detection (Offline)
    num_cells_x, num_cells_y, num_cells_z = len(volume_data), len(volume_data[0]), len(volume_data[0][0])
    F = [[[0 for _ in range(num_cells_z)] for _ in range(num_cells_y)] for _ in range(num_cells_x)]
    
    # Scan volume to detect boundary cells.
    # The sources do not specify the exact boundary detection criterion.
    # A standard implementation checks if scalar values at the 8 corners of the cell
    # span a predefined isovalue threshold [G].
    for i in range(num_cells_x):
        for j in range(num_cells_y):
            for k in range(num_cells_z):
                # is_boundary_cell check would go here
                pass 

    # Step 2: Boundary Cell Projection (Online)
    Z_n = [[float('inf')] * image_width for _ in range(image_height)]
    Z_r = [[float('-inf')] * image_width for _ in range(image_height)]
    
    # Parallel projection setup
    pixel_size = cell_size
    # Incremental vectors for adjacent cells (constant for parallel projection)
    delta_u_i, delta_v_i, delta_d_i = cell_size, 0.0, 0.0
    delta_u_j, delta_v_j, delta_d_j = 0.0, cell_size, 0.0
    delta_u_k, delta_v_k, delta_d_k = 0.0, 0.0, cell_size
    
    u0, v0, d0 = 0.0, 0.0, 0.0
    
    for i in range(num_cells_x):
        for j in range(num_cells_y):
            for k in range(num_cells_z):
                if F[i][j][k] == 1:
                    # Incremental projection using only addition
                    u = u0 + i * delta_u_i + j * delta_u_j + k * delta_u_k
                    v = v0 + i * delta_v_i + j * delta_v_j + k * delta_v_k
                    d = d0 + i * delta_d_i + j * delta_d_j + k * delta_d_k
                    
                    # Cell depth range along view direction
                    dist_front = d - cell_size / 2.0
                    dist_back = d + cell_size / 2.0
                    
                    # Map to pixel coordinates
                    pixel_u = int(u / pixel_size)
                    pixel_v = int(v / pixel_size)
                    
                    if 0 <= pixel_u < image_width and 0 <= pixel_v < image_height:
                        if dist_front < Z_n[pixel_v][pixel_u]:
                            Z_n[pixel_v][pixel_u] = dist_front
                        if dist_back > Z_r[pixel_v][pixel_u]:
                            Z_r[pixel_v][pixel_u] = dist_back

    # Step 3: Ray Traversal
    rendered_image = [[(0.0, 0.0, 0.0)] * image_width for _ in range(image_height)]
    background_color = (0.0, 0.0, 0.0)
    
    for y in range(image_height):
        for x in range(image_width):
            if Z_n[y][x] == float('inf'):
                rendered_image[y][x] = background_color
                continue
                
            t_start = Z_n[y][x]
            t_end = Z_r[y][x]
            
            # Sample and composite along the ray from t_start to t_end.
            # The sources mention adaptive image sampling and early ray termination [S2, unofficial].
            # A standard ray-marching loop is used here [G].
            color = (0.0, 0.0, 0.0)
            t = t_start
            step = cell_size / 2.0
            while t < t_end:
                # Sample volume data at current t
                # Composite color and opacity
                t += step
            rendered_image[y][x] = color
            
    return rendered_image
```

### Parameters
*   **Flag $F$:** Binary flag for each cell $C(i,j,k)$. Value 1 indicates the cell contains the object boundary; 0 indicates empty or fully inside/outside (not a boundary cell) [S2, unofficial].
*   **Cell Size:** The size of the grid cells, which determines the resolution of the boundary estimation [S1, S2, unofficial].
*   **Isovalue:** (Implicit) The scalar value threshold used to define the object boundary within the volume data [G].

### Edge Cases and Pitfalls
*   **Precision:** Incremental projection relies on addition operations, which can accumulate floating-point errors over large volumes [S1, S2, unofficial].
*   **Boundaries:** The algorithm assumes a regular grid of volume elements [S1, S2, unofficial].
*   **Degenerate Input:** If the volume data is empty or contains no boundary cells, the Z-buffers will remain empty, and no rays will be cast [S1, S2, unofficial].

## Subtopics

### Octree Path Computation
Uses octrees to compute ray paths, offering a different hierarchical structure compared to the grid-based boundary cells of BC-RC. Choose when memory efficiency for sparse data is critical. [Octree Path Computation](897-octree-path-computation.md)

### Sparse Voxel Octree Ray Traversal
Traverses sparse voxel octrees, providing better handling of large, sparse datasets than the regular grid approach of BC-RC. Choose for very large, sparse volumes. [Sparse Voxel Octree Ray Traversal](2451-sparse-voxel-octree-ray-traversal.md)

### Rational Ray Parameterization for Octree Traversal
Uses rational parameterization to improve traversal accuracy in octrees, addressing precision issues that may arise in BC-RC's incremental projection. Choose when high precision is required. [Rational Ray Parameterization for Octree Traversal](3875-rational-ray-parameterization-for-octree-traversal.md)

### Ray-Occlusion Traversal in Octree
Focuses on occlusion culling within octrees, which can complement BC-RC's boundary cell skipping. Choose when occlusion is a major performance bottleneck. [Ray-Occlusion Traversal in Octree](5121-ray-occlusion-traversal-in-octree.md)

### Stackless Traversal
Eliminates the need for a stack during traversal, which can be beneficial for GPU implementations, unlike the serial nature of BC-RC. Choose for GPU acceleration. [Stackless Traversal](5202-stackless-traversal.md)

### Presence Acceleration
Accelerates traversal by checking for the presence of data, similar to BC-RC's boundary cell flags but potentially more efficient. Choose for faster empty space skipping. [Presence Acceleration](5753-presence-acceleration.md)

### Octree Raycasting
A general term for ray casting using octrees, which BC-RC improves upon by using boundary cells. Choose when a hierarchical structure is preferred. [Octree Raycasting](11317-octree-raycasting.md)

### Hierarchical Tree Node Traversal
Traverses hierarchical tree nodes, offering a flexible structure compared to the fixed grid of BC-RC. Choose for complex hierarchical data. [Hierarchical Tree Node Traversal](12213-hierarchical-tree-node-traversal.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The algorithm is described as "fast" and improves upon PARC by reducing projection overhead and providing more accurate boundary estimation [S1, S2, unofficial].
*   **Scaling:** Performance scales with the number of boundary cells and viable pixels. Adaptive image sampling can help manage complexity in high-detail regions [S2, unofficial].
*   **Strengths:** Fast serial rendering, accurate boundary estimation, reduced projection overhead via incremental projection, viewpoint-independent preprocessing [S1, S2, unofficial].
*   **Limitations:** Serial algorithm, incremental projection can accumulate floating-point errors over large volumes [S1, S2, unofficial].
*   **When to Use:** Suitable for time-critical volume rendering where accuracy and speed are important [S1, S2, unofficial].
*   **When Not to Use:** Not given in the sources.

## Background
Boundary Cell-Based Ray Casting was proposed as an improvement on the PARC algorithm [S1, S2, unofficial]. It was developed in the context of time-critical volume rendering, aiming to accelerate the ray casting procedure by skipping over empty space in volume datasets [S1, S2, unofficial]. The algorithm was part of a broader effort to establish hierarchical level-of-detail representation of 3D data sets and develop user interfaces for interactive exploration [S1, S2, unofficial].

## Sources
[S1] Time-Critical Volume Rendering, https://ntrs.nasa.gov/api/citations/19990040407/downloads/19990040407.pdf, fetched 2026-10-26 (unofficial)
[S2] NASA Technical Reports Server (NTRS) 19990040407: Time-Critical Volume Rendering, https://archive.org/details/NASA_NTRS_Archive_19990040407, fetched 2026-10-26 (unofficial)
