# Hierarchical Splatting

Also called: Progressive refinement splatting, Pyramid Splatting, Pyramidal volume splatting

## Overview
Hierarchical splatting is a progressive refinement algorithm for volume rendering that uses a pyramidal volume representation to achieve real-time interactive rendering [S1, unofficial]. It solves the performance bottleneck of rendering large, high-resolution volume datasets by decomposing the volume into cubes of varying sizes based on homogeneity, approximating each homogeneous section with a single large splat projected onto the image plane [S3, unofficial]. Unlike standard splatting which treats all voxels equally, hierarchical splatting adapts the resolution of the rendering primitives to the visual complexity of the data, allowing for efficient culling of distant or homogeneous regions while maintaining fidelity in complex areas [S2, unofficial][S4, unofficial].

## How it works
**Inputs:**
*   A 3D volume dataset of size $N^3$ samples [S2, unofficial][S4, unofficial].
*   A user-specified error threshold $\epsilon$ (or precision threshold) that measures the desired homogeneity for the volume decomposition [S1, unofficial][S3, unofficial].

**Outputs:**
*   A rendered image composed of RGBA Gouraud-shaded polygons approximating the splats [S1, unofficial][S3, unofficial].

**Data Structures:**
The core data structure is a **Volume Pyramid** [S2, unofficial][S4, unofficial].
*   **Structure:** A sequence of $\log_2 N$ volumes [S2, unofficial][S4, unofficial].
*   **Level 0:** The original dataset of size $N^3$ [S2, unofficial][S4, unofficial].
*   **Level $k$ ($k > 0$):** Created from Level $k-1$ by averaging each $2 \times 2 \times 2$ group of samples. This results in a volume with one-eighth the resolution (linear dimensions halved) of the previous level [S2, unofficial][S4, unofficial].
*   **Node Content:** Each node (voxel) in the pyramid stores:
    *   Average RGB color [S3, unofficial].
    *   Average Alpha (opacity) [S3, unofficial].
    *   Estimated Error ($E$): The approximation cost of representing the node's children with the node's aggregate properties [S1, unofficial][S3, unofficial]. This error quantifies the deviation if the region is rendered with one average splat instead of the original data [S3, unofficial].

**Algorithm Steps:**

1.  **Pyramid Construction:**
    Build the pyramid from the input volume. For each level $k$ from 1 to $\log_2 N$:
    *   Iterate through all $2 \times 2 \times 2$ blocks of voxels in Level $k-1$.
    *   Compute the average RGB and Alpha for the block to form the voxel at the corresponding position in Level $k$ [S2, unofficial][S4, unofficial].
    *   Compute the error $E$ for the new voxel. The error represents the variance or deviation of the 8 child values from the parent's average [S1, unofficial][S3, unofficial].

2.  **Octree Fitting (Traverse and Select):**
    Traverse the pyramid to determine which nodes are rendered as single splats and which are subdivided.
    *   Start at the root (Level $\log_2 N$).
    *   For a given node, compare its error $E$ against the user threshold $\epsilon$ [S1, unofficial][S3, unofficial].
    *   **If $E \le \epsilon$:** The node is homogeneous enough. It is selected as a leaf node. A single splat is generated for this node, scaled to match the projection of the corresponding volume cell [S1, unofficial][S3, unofficial].
    *   **If $E > \epsilon$:** The node is too heterogeneous. Recursively traverse its 8 children in Level $k-1$ [S1, unofficial][S3, unofficial].
    *   *Note:* The traversal stops at the base level (Level 0) if the threshold is not met, ensuring all data is represented [S3, unofficial].

3.  **Footprint Table Generation:**
    For every level of the pyramid, calculate proper-sized footprint tables [S3, unofficial]. These tables define the screen-space footprint (polygon shape and size) for projecting and compositing cells of that specific level [S3, unofficial]. The footprint size scales with the resolution of the pyramid level; higher levels (coarser resolution) have larger footprints [S1, unofficial].

4.  **Rendering (Splatting):**
    *   Identify the set of leaf nodes from the traversal step.
    *   Sort these leaf nodes in back-to-front order based on the viewing direction [S2, unofficial][S4, unofficial].
    *   For each leaf node:
        *   Project the volume cell onto the image plane.
        *   Approximate the splat as an RGBA Gouraud-shaded polygon [S1, unofficial][S3, unofficial].
        *   Use the pre-calculated footprint table for the node's pyramid level to determine the polygon's geometry [S3, unofficial].
        *   Rasterize the polygon using standard graphics hardware [S1, unofficial][S2, unofficial][S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_pyramid(volume_data):
    """
    Constructs the volume pyramid.
    volume_data: 3D array of shape (N, N, N) containing RGB and Alpha.
    Returns: List of pyramid levels, where level 0 is original data.
    """
    pyramid = [volume_data]
    current_level = volume_data
    n = current_level.shape[0]
    
    # Precompute error for all nodes except the base level
    # Error is stored alongside RGB/Alpha in a structured format or separate array
    # For simplicity, we assume a function compute_error returns the variance metric
    while n > 1:
        next_n = n // 2
        next_level = np.zeros((next_n, next_n, next_n, 4)) # 4 channels: R, G, B, A
        next_error = np.zeros((next_n, next_n, next_n))
        
        for x in range(next_n):
            for y in range(next_n):
                for z in range(next_n):
                    # Extract 2x2x2 block from current level
                    block = current_level[2*x:2*x+2, 2*y:2*y+2, 2*z:2*z+2]
                    
                    # Compute average RGB and Alpha
                    avg_rgb_alpha = np.mean(block, axis=(0, 1, 2))
                    next_level[x, y, z] = avg_rgb_alpha
                    
                    # Compute error (variance of children from parent mean)
                    # [S1, unofficial][S3, unofficial]
                    error = np.mean((block - avg_rgb_alpha) ** 2)
                    next_error[x, y, z] = error
        
        # Store error with the level for traversal
        pyramid.append({'data': next_level, 'error': next_error})
        current_level = next_level
        n = next_n
        
    return pyramid

def traverse_and_select(pyramid, threshold):
    """
    Traverses the pyramid to select leaf nodes for splatting.
    pyramid: List of pyramid levels (from coarse to fine, index 0 is coarsest)
    threshold: User-specified error threshold epsilon
    Returns: List of leaf nodes, each containing (level_index, x, y, z, rgb, alpha)
    """
    leaves = []
    num_levels = len(pyramid)
    
    # Start traversal from the root (coarsest level, index 0)
    # Note: The pyramid list in build_pyramid goes from fine to coarse if appended sequentially?
    # Let's assume pyramid[0] is finest (original) and pyramid[-1] is coarsest (root)
    # Based on S2/S4: "sequence of log(N) volumes... first volume is original... higher levels created from lower"
    # So pyramid[0] = Original (Fine), pyramid[-1] = Root (Coarse).
    # Traversal starts at Root.
    
    def recurse(level_idx, x, y, z):
        node = pyramid[level_idx]
        error = node['error'][x, y, z]
        rgb_alpha = node['data'][x, y, z]
        
        # If error is below threshold, select this node as a leaf
        if error <= threshold:
            leaves.append({
                'level': level_idx,
                'x': x, 'y': y, 'z': z,
                'rgb': rgb_alpha[:3],
                'alpha': rgb_alpha[3]
            })
        else:
            # Recurse to children (finer level)
            # Children are at level_idx + 1
            # Each parent (x,y,z) has 8 children at (2x,2y,2z) to (2x+1, 2y+1, 2z+1)
            if level_idx + 1 < len(pyramid):
                base_x = 2 * x
                base_y = 2 * y
                base_z = 2 * z
                for dx in range(2):
                    for dy in range(2):
                        for dz in range(2):
                            recurse(level_idx + 1, base_x + dx, base_y + dy, base_z + dz)
            else:
                # Reached base level (original data) but error still high
                # Force selection of base level voxels
                leaves.append({
                    'level': level_idx,
                    'x': x, 'y': y, 'z': z,
                    'rgb': rgb_alpha[:3],
                    'alpha': rgb_alpha[3]
                })

    # Start at root
    root_level = len(pyramid) - 1
    root_size = pyramid[root_level]['data'].shape[0]
    for x in range(root_size):
        for y in range(root_size):
            for z in range(root_size):
                recurse(root_level, x, y, z)
                
    return leaves

def render(pyramid, leaves, view_transform, footprint_tables):
    """
    Renders the selected leaves using splatting.
    """
    # Sort leaves back-to-front based on view transform
    sorted_leaves = sort_back_to_front(leaves, view_transform)
    
    # Initialize image buffer
    image = np.zeros((height, width, 4))
    
    for leaf in sorted_leaves:
        level_idx = leaf['level']
        x, y, z = leaf['x'], leaf['y'], leaf['z']
        rgb = leaf['rgb']
        alpha = leaf['alpha']
        
        # Get footprint polygon for this level
        # Footprint size depends on the resolution of the pyramid level
        polygon = footprint_tables[level_idx]
        
        # Project polygon to screen space
        screen_polygon = project_polygon(polygon, view_transform, x, y, z)
        
        # Rasterize polygon with Gouraud shading
        # Color is constant per splat (average of children), but can be interpolated
        # across the polygon if vertex colors were stored (here simplified to flat/average)
        rasterize_polygon(screen_polygon, rgb, alpha, image)
        
    return image
```

**Parameters:**
*   **Error Threshold ($\epsilon$):** A user-specified value determining the desired homogeneity [S3, unofficial]. Selecting a larger error results in fewer subvolumes (coarser decomposition), while decreasing the error increases the number of subvolumes (finer decomposition) [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The error metric computation must be stable. If the volume data has high dynamic range, the error values may require normalization [G].
*   **Boundary Conditions:** The pyramid construction assumes $N$ is a power of 2. If $N$ is not a power of 2, padding or truncation is required [G].
*   **Degenerate Input:** If the volume is completely homogeneous, the root node will have zero error, and only one large splat will be rendered [S3, unofficial]. If the volume is completely random, the algorithm will descend to the base level, rendering $N^3$ splats, losing the performance benefit [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Pyramid construction is $O(N^3)$ [S2, unofficial][S4, unofficial]. Traversal is proportional to the number of leaf nodes, which depends on $\epsilon$. Rendering is proportional to the number of leaf nodes.
*   **Memory Complexity:** The pyramid stores $\sum_{k=0}^{\log N} (N/2^k)^3 \approx \frac{8}{7} N^3$ samples, so memory usage is roughly $1.14 \times$ the original volume size [S2, unofficial][S4, unofficial].
*   **Measured Performance:** For binary data, bits can be packed into bytes to produce lower resolution images faster (1/8th time for half linear resolution) [S2, unofficial]. The algorithm enables real-time interactive rendering suitable for applications requiring progressive refinement [S1, unofficial].
*   **Strengths:** Real-time performance, adaptive resolution, efficient use of graphics hardware via polygon approximation [S1, unofficial][S2, unofficial].
*   **Limitations:** Visual artifacts at boundaries between different pyramid levels if not blended properly (though Gouraud shading helps) [S1, unofficial]. The quality is dependent on the choice of $\epsilon$ [S3, unofficial].

## Background
Hierarchical splatting was introduced by David Laur and Pat Hanrahan in 1991 [S1, unofficial]. It was designed as a progressive refinement algorithm for volume rendering, leveraging the pyramidal volume representation to achieve real-time interactive rendering [S1, unofficial]. The method approximates splats with RGBA Gouraud-shaded polygons to enable efficient rasterization on modern graphics workstations of the time [S1, unofficial]. It is a foundational technique in the family of multi-resolution volume rendering methods [S2, unofficial].

## Sources
[S1] Hierarchical splatting, https://www.researchgate.net/publication/48546966_EWA_volume_splatting, fetched 2026-10-28 (unofficial)
[S2] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S3] Hierarchical Splatting, https://www.cg.tuwien.ac.at/studentwork/VisFoSe98/bernd/hsplat.htm, fetched 2026-10-27 (unofficial)
[S4] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
