# Hierarchical Spatial Enumeration Optimization

Also called: Complete octree representation, Heirarchical Spatial Enumeration, Hierarchical Spatial Enumeration (Octree Pyramid), Pyramid Enumeration, Pyramid of binary volumes

## Overview
Hierarchical Spatial Enumeration Optimization is an acceleration technique for image-order volume rendering that encodes spatial coherence (empty regions) using a pyramid of binary volumes [S1][S2, unofficial][S3, unofficial]. It solves the problem of inefficient traversal through large, sparse volumetric datasets by allowing rays to skip large empty regions in logarithmic time relative to the ray length [S2, unofficial][S3, unofficial]. Unlike condensed octree representations or object-order traversal methods, this method uses a complete octree structure stored as a pyramid of binary volumes, enabling efficient image-order ray tracing [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset of size $N \times N \times N$ voxels, where $N = 2^M + 1$ for some integer $M \ge 0$ [S1][S3, unofficial]. The dataset consists of opacity values $\alpha(\mathbf{v})$ for each voxel $\mathbf{v}$ [S1].
*   **Output:** A pyramid of $M+1$ binary volumes $V_m$, where $V_m(\mathbf{i}) \in \{0, 1\}$ indicates whether the region covered by cell $\mathbf{i}$ at level $m$ contains any non-zero opacity [S1][S3, unofficial].

**Data Structures:**
The data structure is a pyramid of $M+1$ binary volumes indexed by level $m \in \{0, \dots, M\}$ [S1][S3, unofficial].
*   **Volume $V_m$:** A 3D array of size $C_m \times C_m \times C_m$, where $C_m = (N-1)/2^m$ [S1][S3, unofficial].
*   **Cell Index $\mathbf{i}$:** A vector $(i, j, k)$ where $i, j, k \in \{1, \dots, N-1\}$ [S1][S3, unofficial].
*   **Cell Value $V_m(\mathbf{i})$:**
    *   $1$: The cell contains at least one voxel with non-zero opacity (occupied) [S1][S3, unofficial].
    *   $0$: All voxels in the cell have zero opacity (empty) [S1][S3, unofficial].
*   **Geometry:** Voxels are treated as point samples. Cells fill the space between voxels. The size of a cell on level $m$ is $2^m$ times the spacing between voxels [S1][S3, unofficial]. Each volume $V_m$ is one cell larger in each direction than the underlying dataset it represents at that resolution [S1][S3, unofficial].
*   **Coordinate Alignment:** Voxel $(1,1,1)$ is placed at the front-lower-right corner of cell $(1,1,1)$ on level 0. Cell $(1,1,1)$ on level 0 encloses the space between voxels $(1,1,1)$ and $(2,2,2)$ [S1].

**Construction Algorithm:**
The pyramid is constructed recursively from the base level $V_0$ up to the top level $V_M$ [S1][S3, unofficial].
1.  **Base Level ($V_0$):** For each cell $\mathbf{i} = (i,j,k)$ in $V_0$, check the 8 voxels at its vertices [S1]. A cell is occupied if any of its 8 corner voxels has non-zero opacity [S1][S3, unofficial].
    $$
    V_0(\mathbf{i}) = \begin{cases} 
    1 & \text{if } \alpha(\mathbf{i} + \Delta\mathbf{i}) > 0 \text{ for any } \Delta\mathbf{i} \in \{0, 1\}^3 \\
    0 & \text{otherwise}
    \end{cases}
    $$
    where $\mathbf{i} \in \{1, \dots, N-1\}^3$ [S1][S3, unofficial].
2.  **Recursive Levels ($m > 0$):** For each cell $\mathbf{i}$ in volume $V_m$, check the 8 cells in volume $V_{m-1}$ that form its octants [S1]. A cell is occupied if any of its 8 child cells is occupied [S1][S3, unofficial].
    $$
    V_m(\mathbf{i}) = \max_{\Delta\mathbf{i} \in \{0, 1\}^3} V_{m-1}(2\mathbf{i} - \mathbf{1} + \Delta\mathbf{i})
    $$
    (Indices are 1-based as defined in the source; the formula aggregates the 8 sub-cells from the previous level) [S1][S3, unofficial].

**Ray Traversal Algorithm:**
For each pixel, a viewing ray is cast. The traversal proceeds as follows:
1.  **Entry Point:** Compute the entry point of the ray into the single cell at the top level ($V_M$) [S3, unofficial].
2.  **Traversal Loop:** While the ray has not exited the volume:
    a.  **Test Current Cell:** Let $\mathbf{c}$ be the current cell at level $m$ with value $V_m(\mathbf{c})$ [S3, unofficial].
    b.  **Empty Cell ($V_m(\mathbf{c}) = 0$):**
        i.  Advance the ray parameter $t$ to the exit of the current cell $\mathbf{c}$ [S3, unofficial].
        ii. Determine the next cell $\mathbf{c}'$ on the same level $m$ that the ray enters [S3, unofficial].
        iii. If the parent of $\mathbf{c}'$ differs from the parent of $\mathbf{c}$, move up to the parent level (increment $m$) and repeat the test for the new cell. This allows quick advancement across empty regions [S3, unofficial].
    c.  **Occupied Cell ($V_m(\mathbf{c}) = 1$):**
        i.  If $m > 0$, move down one level ($m \leftarrow m-1$) to the child cell that encloses the current ray position [S3, unofficial].
        ii. If $m = 0$ (leaf level), the cell contains non-zero opacity [S3, unofficial]. Draw samples at evenly spaced locations along the ray segment within this cell [S3, unofficial]. Resample the volume data at these locations and composite the color and opacity into the ray state (front-to-back compositing) [S3, unofficial].
3.  **Termination:** The ray tracing for a pixel terminates when the ray exits the volume or when an adaptive termination criterion is met (e.g., accumulated opacity exceeds a threshold) [S1][S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def construct_pyramid(voxels, N):
    """
    Construct the hierarchical spatial enumeration pyramid.
    voxels: 3D list of size N x N x N containing opacity values.
    N: Side length of the voxel grid (N = 2^M + 1).
    Returns: List of volumes [V_0, V_1, ..., V_M].
    """
    M = int(math.log2(N - 1))
    pyramid = []
    size = N - 1
    # V_prev represents V_{m-1}, initially V_0
    V_prev = [[[0]*size for _ in range(size)] for _ in range(size)]
    
    for i in range(size):
        for j in range(size):
            for k in range(size):
                occupied = False
                for di in (0, 1):
                    for dj in (0, 1):
                        for dk in (0, 1):
                            if voxels[i+di][j+dj][k+dk] > 0:
                                occupied = True
                                break
                        if occupied: break
                    if occupied: break
                V_prev[i][j][k] = 1 if occupied else 0
    pyramid.append(V_prev)
    
    for m in range(1, M + 1):
        prev_size = len(V_prev)
        curr_size = prev_size // 2
        V_curr = [[[0]*curr_size for _ in range(curr_size)] for _ in range(curr_size)]
        for i in range(curr_size):
            for j in range(curr_size):
                for k in range(curr_size):
                    occupied = False
                    for di in (0, 1):
                        for dj in (0, 1):
                            for dk in (0, 1):
                                if V_prev[2*i+di][2*j+dj][2*k+dk] == 1:
                                    occupied = True
                                    break
                            if occupied: break
                        if occupied: break
                    V_curr[i][j][k] = 1 if occupied else 0
        pyramid.append(V_curr)
        V_prev = V_curr
    return pyramid

def ray_box_intersection(origin, direction, min_coord, max_coord):
    """Returns (t_min, t_max) for intersection with axis-aligned box."""
    t_min = -float('inf')
    t_max = float('inf')
    for d in range(3):
        if abs(direction[d]) < 1e-8:
            if origin[d] < min_coord[d] or origin[d] > max_coord[d]:
                return (-1.0, -1.0)
        else:
            t1 = (min_coord[d] - origin[d]) / direction[d]
            t2 = (max_coord[d] - origin[d]) / direction[d]
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
    return (t_min, t_max)

def compute_next_cell(cell, direction):
    """Simple DDA step to next cell boundary."""
    next_cell = list(cell)
    for d in range(3):
        if direction[d] > 0:
            next_cell[d] += 1
        elif direction[d] < 0:
            next_cell[d] -= 1
    return tuple(next_cell)

def compute_parent(cell):
    """Compute parent cell index (0-based)."""
    return tuple(c // 2 for c in cell)

def compute_child_cell(parent_cell, ray_pos, parent_size, child_size):
    """Find child cell containing ray position."""
    child = []
    for d in range(3):
        offset = ray_pos[d] - parent_cell[d] * parent_size
        child.append(parent_cell[d] * 2 + int(offset / child_size))
    return tuple(child)

def sample_volume(voxels, pos):
    """Trilinear interpolation placeholder."""
    # In practice, interpolate from 8 surrounding voxels.
    return 0.0

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
    # Top level is a single cell covering [0, N-1] in cell coordinates
    t_min, t_max = ray_box_intersection(ray_origin, ray_direction, (0,0,0), (N-1, N-1, N-1))
    if t_max < 0: return (0.0, 0.0)
    
    t = t_min
    curr_level = M
    curr_cell = (0, 0, 0)
    acc_color = (0.0, 0.0, 0.0)
    acc_opacity = 0.0
    
    while t < t_max:
        cell_val = pyramid[curr_level][curr_cell[0]][curr_cell[1]][curr_cell[2]]
        
        if cell_val == 0:
            # Empty: advance to next cell on same level
            cell_size = 2 ** curr_level
            t_exit = ray_box_intersection(ray_origin, ray_direction, 
                                          (curr_cell[0]*cell_size, curr_cell[1]*cell_size, curr_cell[2]*cell_size),
                                          ((curr_cell[0]+1)*cell_size, (curr_cell[1]+1)*cell_size, (curr_cell[2]+1)*cell_size))[1]
            if t_exit > t_max: t_exit = t_max
            t = t_exit
            
            # Determine next cell
            new_cell = compute_next_cell(curr_cell, ray_direction)
            new_parent = compute_parent(new_cell)
            old_parent = compute_parent(curr_cell)
            
            if new_parent != old_parent:
                curr_level += 1
                curr_cell = new_parent
            else:
                curr_cell = new_cell
        else:
            # Occupied
            if curr_level == 0:
                # Leaf: sample within cell
                cell_size = 1
                t_cell_min, t_cell_max = ray_box_intersection(ray_origin, ray_direction, 
                                                              (curr_cell[0], curr_cell[1], curr_cell[2]),
                                                              (curr_cell[0]+1, curr_cell[1]+1, curr_cell[2]+1))
                num_samples = 8
                for s in range(num_samples):
                    t_sample = t_cell_min + (t_cell_max - t_cell_min) * (s + 0.5) / num_samples
                    if t_sample > t_max: break
                    pos = ray_origin + t_sample * ray_direction
                    alpha = sample_volume(voxels, pos)
                    # Front-to-back compositing
                    weight = alpha * (1.0 - acc_opacity)
                    acc_color = acc_color + weight * (1.0, 1.0, 1.0) # Placeholder color
                    acc_opacity += weight
                t = t_cell_max
                new_cell = compute_next_cell(curr_cell, ray_direction)
                new_parent = compute_parent(new_cell)
                old_parent = compute_parent(curr_cell)
                if new_parent != old_parent:
                    curr_level += 1
                    curr_cell = new_parent
                else:
                    curr_cell = new_cell
            else:
                # Move down to enclosing child
                curr_level -= 1
                child_size = 2 ** curr_level
                parent_size = 2 * child_size
                curr_cell = compute_child_cell(curr_cell, ray_origin + t * ray_direction, parent_size, child_size)
                
    return (acc_color, acc_opacity)
```

**Parameters:**
*   $M$: Integer such that dataset side length $N = 2^M + 1$. The pyramid has levels $m = 0, \dots, M$ [S1].
*   Cell Size: The size of cells on level $m$ is $2^m$ times the spacing between voxels [S1][S3, unofficial].

## Subtopics

### [Hierarchical Volume Traversal with Adaptive Termination](1682-hierarchical-volume-traversal-with-adaptive-termination.md)
Hierarchical Volume Traversal with Adaptive Termination is a ray tracing algorithm that uses a pyramidal data structure of binary volumes to skip empty regions and an adaptive termination criterion to stop tracing when accumulated opacity exceeds a threshold. Unlike geometric ray tracing, which uses bounding volumes or spatial subdivision with lists of primitives, this method uses a hierarchical binary volume structure, which is more efficient for volume data because volume datasets have tens of millions of spatially ordered cells with single samples, whereas geometric scenes have hundreds of cells with many primitives.

### Hierarchical Opacity Enumeration
An optimization that uses a binary octree to avoid unnecessary resampling of voxels in transparent regions of the volume [S4].

### Flat Pyramid
A hierarchical spatial data structure (quadtree recursively subdivided down to pixel size) where empty space uniformity information is stored in the empty voxels of the 3D volume grid [S5, unofficial]. The structure is a quadtree recursively subdivided down until pixel size. Gray squares denote voxels with volume data; white squares denote empty voxels storing the octree level [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** The cost of ray tracing volumetric data using a hierarchical enumeration grows linearly with the size of the image, logarithmically with the length of each ray, and linearly with the depth complexity of the data [S2, unofficial].
*   **Memory Complexity:** The pyramid requires storing $M+1$ binary volumes [S1]. The total number of cells is $\sum_{m=0}^{M} ((N-1)/2^m)^3$ [G]. For large $N$, this is dominated by the base level $V_0$, which has $(N-1)^3$ cells [G]. Thus, memory usage is $O(N^3)$, same as the original dataset, but with a smaller constant factor (1 bit per cell vs. multiple bits per voxel) [G].
*   **Performance:** Speedups of an order of magnitude have been observed for datasets of useful size and complexity [S1]. The method saves time by quickly advancing across empty regions of space using the binary pyramid structure [S3, unofficial].
*   **Strengths:** Efficiently skips large empty regions; suitable for image-order rendering; simple binary structure [S1].
*   **Limitations:** Requires $O(N^3)$ memory for the pyramid; construction time is non-trivial; less flexible than condensed octrees for highly irregular data [G].
*   **When to use:** When rendering large, sparse volumetric datasets with significant empty regions using image-order rendering [S1].
*   **When not to use:** When memory is constrained and the dataset is dense; when object-order rendering is preferred [G].

## Background
Hierarchical Spatial Enumeration Optimization was introduced by Marc Levoy in the late 1980s as part of his work on efficient volume rendering algorithms [S1][S2, unofficial]. The method is based on the observation that many datasets contain coherent regions of uninteresting voxels (opacity zero) [S2, unofficial]. It differs from earlier methods like [25] (likely referring to a condensed octree representation) by using a pyramid of binary volumes/complete octree rather than a condensed representation, and by rendering in image order rather than object order [S1].

## Sources
[S1] Efficient ray tracing of volume data, https://doi.org/10.1145/78964.78965, fetched 2026-10-25
[S2] DTIC ADA208085: Rendering Mixtures of Geometric and Volumetric Data, https://archive.org/details/DTIC_ADA208085, fetched 2026-10-26 (unofficial)
[S3] Volume Rendering With Ray Casting - WPI, https://web.cs.wpi.edu/~matt/courses/cs563/talks/powwie/p1/ray-cast.htm, fetched 2026-10-26 (unofficial)
[S4] Volume rendering on scalable shared-memory MIMD architectures, https://doi.org/10.1145/147130.147141, fetched 2026-10-25
[S5] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume-Rendering, fetched 2026-10-25 (unofficial)
