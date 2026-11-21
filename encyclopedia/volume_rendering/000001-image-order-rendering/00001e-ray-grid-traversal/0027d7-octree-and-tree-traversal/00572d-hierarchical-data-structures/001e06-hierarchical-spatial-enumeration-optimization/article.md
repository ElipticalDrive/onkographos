# Hierarchical Spatial Enumeration Optimization

Also called: Complete octree representation, Heirarchical Spatial Enumeration, Hierarchical Spatial Enumeration (Octree Pyramid), Pyramid Enumeration, Pyramid of binary volumes

## Overview
Hierarchical Spatial Enumeration Optimization is an acceleration technique for image-order volume rendering that encodes spatial coherence (empty regions) using a pyramid of binary volumes [S1][S2, S3, unofficial]. It solves the problem of inefficient traversal through large, sparse volumetric datasets by allowing rays to skip large empty regions in logarithmic time relative to the ray length [S2, S3, unofficial]. Unlike condensed octree representations or object-order traversal methods, this method uses a complete octree structure stored as a pyramid of binary volumes, enabling efficient image-order ray tracing [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset of size $N \times N \times N$ voxels, where $N = 2^M + 1$ for some integer $M \ge 0$. The dataset consists of opacity values $\alpha(\mathbf{v})$ for each voxel $\mathbf{v}$.
*   **Output:** A pyramid of $M+1$ binary volumes $V_m$, where $V_m(\mathbf{i}) \in \{0, 1\}$ indicates whether the region covered by cell $\mathbf{i}$ at level $m$ contains any non-zero opacity.

**Data Structures:**
The data structure is a pyramid of $M+1$ binary volumes indexed by level $m \in \{0, \dots, M\}$.
*   **Volume $V_m$:** A 3D array of size $C_m \times C_m \times C_m$, where $C_m = (N-1)/2^m$.
*   **Cell Index $\mathbf{i}$:** A vector $(i, j, k)$ where $i, j, k \in \{1, \dots, N-1\}$.
*   **Cell Value $V_m(\mathbf{i})$:**
    *   $1$: The cell contains at least one voxel with non-zero opacity (occupied).
    *   $0$: All voxels in the cell have zero opacity (empty).
*   **Geometry:** Voxels are treated as point samples. Cells fill the space between voxels. The size of a cell on level $m$ is $2^m$ times the spacing between voxels. Each volume $V_m$ is one cell larger in each direction than the underlying dataset it represents at that resolution [S1][S3, unofficial].
*   **Coordinate Alignment:** Voxel $(1,1,1)$ is placed at the front-lower-right corner of cell $(1,1,1)$ on level 0. Cell $(1,1,1)$ on level 0 encloses the space between voxels $(1,1,1)$ and $(2,2,2)$ [S1][S3, unofficial].

**Construction Algorithm:**
The pyramid is constructed recursively from the base level $V_0$ up to the top level $V_M$.

1.  **Base Level ($V_0$):** For each cell $\mathbf{i} = (i,j,k)$ in $V_0$, check the 8 voxels at its vertices. A cell is occupied if any of its 8 corner voxels has non-zero opacity.
    $$
    V_0(\mathbf{i}) = \begin{cases} 
    1 & \text{if } \alpha(\mathbf{i} + \Delta\mathbf{i}) > 0 \text{ for any } \Delta\mathbf{i} \in \{0, 1\}^3 \\
    0 & \text{otherwise}
    \end{cases}
    $$
    where $\mathbf{i} \in \{1, \dots, N-1\}^3$ [S1][S3, unofficial].

2.  **Recursive Levels ($m > 0$):** For each cell $\mathbf{i}$ in volume $V_m$, check the 8 cells in volume $V_{m-1}$ that form its octants. A cell is occupied if any of its 8 child cells is occupied.
    $$
    V_m(\mathbf{i}) = \max_{\Delta\mathbf{i} \in \{0, 1\}^3} V_{m-1}(2\mathbf{i} - \mathbf{1} + \Delta\mathbf{i})
    $$
    (Note: The exact indexing of children depends on the coordinate system, but logically it aggregates the 8 sub-cells from the previous level) [S1][S3, unofficial].

**Ray Traversal Algorithm:**
For each pixel, a viewing ray is cast. The traversal proceeds as follows:

1.  **Entry Point:** Compute the entry point of the ray into the single cell at the top level ($V_M$) [S3, unofficial].
2.  **Traversal Loop:** While the ray has not exited the volume:
    a.  **Test Current Cell:** Let $\mathbf{c}$ be the current cell at level $m$ with value $V_m(\mathbf{c})$.
    b.  **Empty Cell ($V_m(\mathbf{c}) = 0$):**
        i.  Advance the ray parameter $t$ to the exit of the current cell $\mathbf{c}$.
        ii. Determine the next cell $\mathbf{c}'$ on the same level $m$ that the ray enters.
        iii. If the parent of $\mathbf{c}'$ differs from the parent of $\mathbf{c}$, move up to the parent level (increment $m$) and repeat the test for the new cell. This allows quick advancement across empty regions [S3, unofficial].
    c.  **Occupied Cell ($V_m(\mathbf{c}) = 1$):**
        i.  If $m > 0$, move down one level ($m \leftarrow m-1$) to the child cell that encloses the current ray position.
        ii. If $m = 0$ (leaf level), the cell contains non-zero opacity. Draw samples at evenly spaced locations along the ray segment within this cell. Resample the volume data at these locations and composite the color and opacity into the ray state (front-to-back compositing) [S3, unofficial].
3.  **Termination:** The ray tracing for a pixel terminates when the ray exits the volume or when an adaptive termination criterion is met (e.g., accumulated opacity exceeds a threshold) [S1][S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_pyramid(voxels, N):
    """
    Construct the hierarchical spatial enumeration pyramid.
    voxels: 3D array of size N x N x N containing opacity values.
    N: Side length of the voxel grid (N = 2^M + 1).
    Returns: List of volumes [V_0, V_1, ..., V_M].
    """
    M = int(log2(N - 1))
    pyramid = []
    
    # Construct base level V_0
    # V_0 has size (N-1) x (N-1) x (N-1)
    # Cell (i,j,k) in V_0 covers voxels at corners (i,j,k) to (i+1,j+1,k+1)
    # Note: Indices in sources are 1-based, we use 0-based for implementation
    size_0 = N - 1
    V_0 = np.zeros((size_0, size_0, size_0), dtype=np.uint8)
    
    for i in range(size_0):
        for j in range(size_0):
            for k in range(size_0):
                # Check 8 vertices of the cell
                # Vertices are at voxel coordinates (i,j,k) to (i+1,j+1,k+1)
                occupied = False
                for di in [0, 1]:
                    for dj in [0, 1]:
                        for dk in [0, 1]:
                            # Check if voxel exists and has opacity > 0
                            if 0 <= i+di < N and 0 <= j+dj < N and 0 <= k+dk < N:
                                if voxels[i+di, j+dj, k+dk] > 0:
                                    occupied = True
                                    break
                        if occupied: break
                    if occupied: break
                V_0[i, j, k] = 1 if occupied else 0
    pyramid.append(V_0)
    
    # Construct higher levels
    current_vol = V_0
    for m in range(1, M + 1):
        prev_vol = current_vol
        prev_size = prev_vol.shape[0]
        curr_size = prev_size // 2
        V_m = np.zeros((curr_size, curr_size, curr_size), dtype=np.uint8)
        
        for i in range(curr_size):
            for j in range(curr_size):
                for k in range(curr_size):
                    # Check 8 children in the previous level
                    # Children are at indices (2*i, 2*j, 2*k) to (2*i+1, 2*j+1, 2*k+1)
                    occupied = False
                    for di in [0, 1]:
                        for dj in [0, 1]:
                            for dk in [0, 1]:
                                if prev_vol[2*i+di, 2*j+dj, 2*k+dk] == 1:
                                    occupied = True
                                    break
                            if occupied: break
                        if occupied: break
                    V_m[i, j, k] = 1 if occupied else 0
        pyramid.append(V_m)
        current_vol = V_m
        
    return pyramid

def traverse_pyramid(ray_origin, ray_direction, pyramid, N):
    """
    Traverse the pyramid to render a single ray.
    ray_origin: 3D vector
    ray_direction: 3D vector
    pyramid: List of volumes [V_0, ..., V_M]
    N: Side length of the voxel grid
    Returns: Composite color and opacity
    """
    M = len(pyramid) - 1
    # Cell size at level m is 2^m * voxel_spacing
    # Assume voxel spacing is 1 for simplicity, so cell size is 2^m
    # The volume bounds are from 0 to N-1 in cell coordinates for V_0
    
    # Compute entry into top level cell V_M
    # V_M is a single cell covering the entire volume
    # Ray entry point in cell coordinates
    t_min, t_max = compute_ray_box_intersection(ray_origin, ray_direction, 
                                                box_min=[0,0,0], box_max=[N-1, N-1, N-1])
    
    if t_max < 0:
        return (0.0, 0.0) # Ray misses volume
        
    t = t_min
    accumulated_opacity = 0.0
    accumulated_color = (0.0, 0.0, 0.0)
    
    # Current cell coordinates in V_M
    # Since V_M is 1x1x1, current cell is (0,0,0)
    curr_cell = (0, 0, 0)
    curr_level = M
    
    while t < t_max:
        # Get cell value
        cell_val = pyramid[curr_level][curr_cell]
        
        if cell_val == 0:
            # Empty cell: advance to next cell on same level
            # Compute exit from current cell
            cell_size = 2 ** curr_level
            # Local coordinates of current cell
            x0, y0, z0 = curr_cell
            # Global bounds of current cell in voxel units
            # Cell (i,j,k) at level m covers [i*2^m, (i+1)*2^m]
            # But note: V_0 cells cover space between voxels.
            # The geometry is complex. Simplified: advance t to exit of current cell.
            
            # Compute exit t for current cell
            t_exit = compute_ray_exit_t(ray_origin, ray_direction, 
                                        box_min=[x0*cell_size, y0*cell_size, z0*cell_size],
                                        box_max=[(x0+1)*cell_size, (y0+1)*cell_size, (z0+1)*cell_size])
            
            if t_exit > t_max:
                t_exit = t_max
                
            t = t_exit
            
            # Determine next cell
            # This requires computing the new cell index based on ray direction
            # If parent changes, move up
            new_cell = compute_next_cell(curr_cell, ray_direction, cell_size)
            new_parent = compute_parent(new_cell)
            old_parent = compute_parent(curr_cell)
            
            if new_parent != old_parent:
                curr_level += 1
                curr_cell = new_parent
            else:
                curr_cell = new_cell
        else:
            # Occupied cell
            if curr_level == 0:
                # Leaf level: sample within this cell
                cell_size = 1 # V_0 cell size in terms of voxel spacing? 
                # Actually, V_0 cells are size 1 in "cell units", but cover space between voxels.
                # The ray segment within this cell needs to be sampled.
                
                # Compute entry and exit t for this cell
                x0, y0, z0 = curr_cell
                # Cell bounds in voxel space
                # Cell (i,j,k) in V_0 encloses space between voxels (i,j,k) and (i+1,j+1,k+1)
                # So bounds are [i, i+1], [j, j+1], [k, k+1]
                t_cell_min, t_cell_max = compute_ray_box_intersection(ray_origin, ray_direction,
                                                                      box_min=[x0, y0, z0],
                                                                      box_max=[x0+1, y0+1, k+1])
                
                # Sample along this segment
                num_samples = 8 # Or based on cell size
                for s in range(num_samples):
                    t_sample = t_cell_min + (t_cell_max - t_cell_min) * (s + 0.5) / num_samples
                    if t_sample > t_max: break
                    
                    # Get world position
                    pos = ray_origin + t_sample * ray_direction
                    
                    # Sample volume data (trilinear interpolation)
                    alpha = sample_volume(voxels, pos)
                    
                    # Composite
                    # C_out = C_in + alpha * (1 - accumulated_opacity) * color
                    # accumulated_opacity += alpha * (1 - accumulated_opacity)
                    # (Simplified compositing)
                    pass
                
                # Move to next cell
                t = t_cell_max
                new_cell = compute_next_cell(curr_cell, ray_direction, 1)
                new_parent = compute_parent(new_cell)
                old_parent = compute_parent(curr_cell)
                
                if new_parent != old_parent:
                    curr_level += 1
                    curr_cell = new_parent
                else:
                    curr_cell = new_cell
            else:
                # Move down one level
                curr_level -= 1
                # Find child cell that contains current ray position
                # This requires mapping the current ray position to the child cell index
                # Since we are inside an occupied cell, we move down to the specific child
                # that contains the ray.
                # The ray position is within the current cell.
                # We need to find which of the 8 children contains the ray.
                # This is typically done by comparing ray position relative to cell center.
                # However, since we are traversing, we can compute the child index from the parent index and ray direction.
                # For simplicity, assume we compute the child index based on the ray's position within the parent cell.
                # This is a placeholder for the specific child selection logic.
                curr_cell = compute_child_cell(curr_cell, ray_direction, 2 ** (curr_level + 1))
                
    return (accumulated_color, accumulated_opacity)
```

**Parameters:**
*   $M$: Integer such that dataset side length $N = 2^M + 1$. The pyramid has levels $m = 0, \dots, M$ [S1].
*   Cell Size: The size of cells on level $m$ is $2^m$ times the spacing between voxels [S1][S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Coordinate Alignment:** Voxel $(1,1,1)$ is placed at the front-lower-right corner of cell $(1,1,1)$ on level 0. Cell $(1,1,1)$ on level 0 encloses the space between voxels $(1,1,1)$ and $(2,2,2)$ [S1][S3, unofficial]. Care must be taken to align ray coordinates with this cell-based indexing.
*   **Precision:** Floating-point precision issues may arise when computing ray-cell intersections, especially at higher levels where cell sizes are large.
*   **Boundary Conditions:** Rays entering or exiting the volume must be handled correctly, especially when moving up the pyramid levels.

## Subtopics

### [Hierarchical Volume Traversal with Adaptive Termination](1682-hierarchical-volume-traversal-with-adaptive-termination.md)
Hierarchical Volume Traversal with Adaptive Termination is a ray tracing algorithm that uses a pyramidal data structure of binary volumes to skip empty regions and an adaptive termination criterion to stop tracing when accumulated opacity exceeds a threshold. Unlike geometric ray tracing, which uses bounding volumes or spatial subdivision with lists of primitives, this method uses a hierarchical binary volume structure, which is more efficient for volume data because volume datasets have tens of millions of spatially ordered cells with single samples, whereas geometric scenes have hundreds of cells with many primitives.

### Hierarchical Opacity Enumeration
An optimization that uses a binary octree to avoid unnecessary resampling of voxels in transparent regions of the volume [S4].

### Flat Pyramid
A hierarchical spatial data structure (quadtree recursively subdivided down to pixel size) where empty space uniformity information is stored in the empty voxels of the 3D volume grid [S5, unofficial]. The structure is a quadtree recursively subdivided down until pixel size. Gray squares denote voxels with volume data; white squares denote empty voxels storing the octree level [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** The cost of ray tracing volumetric data using a hierarchical enumeration grows linearly with the size of the image, logarithmically with the length of each ray, and linearly with the depth complexity of the data [S2, unofficial].
*   **Memory Complexity:** The pyramid requires storing $M+1$ binary volumes. The total number of cells is $\sum_{m=0}^{M} ((N-1)/2^m)^3$. For large $N$, this is dominated by the base level $V_0$, which has $(N-1)^3$ cells. Thus, memory usage is $O(N^3)$, same as the original dataset, but with a smaller constant factor (1 bit per cell vs. multiple bits per voxel).
*   **Performance:** Speedups of an order of magnitude have been observed for datasets of useful size and complexity [S1]. The method saves time by quickly advancing across empty regions of space using the binary pyramid structure [S3, unofficial].
*   **Strengths:** Efficiently skips large empty regions; suitable for image-order rendering; simple binary structure.
*   **Limitations:** Requires $O(N^3)$ memory for the pyramid; construction time is non-trivial; less flexible than condensed octrees for highly irregular data.
*   **When to use:** When rendering large, sparse volumetric datasets with significant empty regions using image-order rendering.
*   **When not to use:** When memory is constrained and the dataset is dense; when object-order rendering is preferred.

## Background
Hierarchical Spatial Enumeration Optimization was introduced by Marc Levoy in the late 1980s as part of his work on efficient volume rendering algorithms [S1][S2, unofficial]. It was one of the first techniques to exploit spatial coherence in volumetric data for acceleration [S2, unofficial]. The method is based on the observation that many datasets contain coherent regions of uninteresting voxels (opacity zero) [S2, unofficial]. It differs from earlier methods like [25] (likely referring to a condensed octree representation) by using a pyramid of binary volumes/complete octree rather than a condensed representation, and by rendering in image order rather than object order [S1].

## Sources
[S1] Efficient ray tracing of volume data, https://doi.org/10.1145/78964.78965, fetched 2026-10-25
[S2] DTIC ADA208085: Rendering Mixtures of Geometric and Volumetric Data, https://archive.org/details/DTIC_ADA208085, fetched 2026-10-26 (unofficial)
[S3] Volume Rendering With Ray Casting - WPI, https://web.cs.wpi.edu/~matt/courses/cs563/talks/powwie/p1/ray-cast.htm, fetched 2026-10-26 (unofficial)
[S4] Volume rendering on scalable shared-memory MIMD architectures, https://doi.org/10.1145/147130.147141, fetched 2026-10-25
[S5] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume-Rendering, fetched 2026-10-25 (unofficial)
