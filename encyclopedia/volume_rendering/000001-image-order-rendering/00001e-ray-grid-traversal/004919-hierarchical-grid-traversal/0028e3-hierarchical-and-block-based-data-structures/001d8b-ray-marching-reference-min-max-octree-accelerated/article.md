# Ray-marching reference (min-max octree accelerated)

Also called: Min-Max Octree Ray Casting, Min-Max Octree Ray-Marching, Min-Max Octree Ray-Marching Reference, Knoll et al. octree ray tracing, Interactive Isosurface Ray Tracing (Knoll et al.)

## Overview
This method is an accelerated direct isosurface visualization technique that uses a min-max octree to facilitate empty space skipping during ray marching [S1][S3, unofficial][S2]. It serves as a performance baseline for general volume rendering, offering negligible memory overhead for the input volume and low auxiliary data requirements (1–4 MB for octree depth $\ge$ 7) [S3, unofficial][S2]. Unlike Sphere Tracing or Segment Tracing, which require signed distance functions or are limited to hierarchical skeletal implicit surfaces, this approach is applicable to a wide range of scalar volumes [S2][S3, unofficial]. It guarantees continuous isosurfaces that locally interpolate the input data, unlike mesh extraction techniques (e.g., marching cubes) which may produce topologically different piecewise linear meshes [S4, unofficial].

## How it works
**Inputs:**
*   Scalar volume data $V$ defined on a 3D grid.
*   Isosurface threshold $T$.
*   Camera parameters (origin $O$, direction $D$ for each pixel).

**Outputs:**
*   Rendered image with isosurface shading.

**Data Structures:**
The acceleration structure is a min-max octree. The octree is constructed on a base level with equal size along all three cardinal axes [S1][S3, unofficial].
*   **Node Layout:** Each node stores the minimum ($min$) and maximum ($max$) scalar values of the scalar field within its subtree [G].
*   **Construction:** The octree can be built directly around voxels [S5, unofficial] or derived from a MIP texture created from a min-max texture used for block occupancy determination [S1][S3, unofficial]. For straightforward implementation, volume dimensions are padded to the nearest power of two [S1].
*   **Compression:** In some implementations (Knoll et al.), the octree is embedded with scalar data in a single structure, allowing lossless compression into 10–30% of the original size [S4, S5, unofficial].

**Algorithm Steps:**

1.  **Octree Construction:**
    Build the min-max octree from the input volume. If using the MIP texture approach, generate the MIP levels from the min-max texture [S1][S3, unofficial]. If using the Knoll et al. approach, convert the 3D grid into a lossless-compression octree representation [S4, unofficial].

2.  **Ray Traversal (Empty Space Skipping):**
    For each ray, traverse the octree from root to leaf.
    *   If a node's range $[min, max]$ does not contain the isosurface threshold $T$ (i.e., $T < min$ or $T > max$), the node is pruned (empty space skipping) [G].
    *   If the node contains $T$, traverse its children.
    *   The traversal follows the hierarchy until leaf nodes are reached or the ray exits the volume [S3, S5, unofficial].

3.  **Ray Marching:**
    March through the volume within the relevant octree nodes.
    *   **Step Size:** Use a step size $s = 0.5 \times \text{cell edge length}$ [S1][S3, unofficial].
    *   **Sampling:** Sample the scalar field at the current ray position.

4.  **Isosurface Intersection:**
    When a sample crosses the threshold $T$ (sign change detected):
    *   **Refinement:** Perform exactly three iterations of a binary search algorithm to refine the intersection position [S1][S3, unofficial].
    *   **Normal Calculation:** Compute the surface normal on the fly using the central difference method [S1][S3, unofficial].

5.  **Shading:**
    Apply Gooch shading using the computed vertex normals [S1][S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_min_max_octree(ray_origin, ray_direction, volume, isovalue, octree):
    """
    Ray march through a volume using a min-max octree for acceleration.
    
    Args:
        ray_origin: (x, y, z) camera origin.
        ray_direction: (x, y, z) normalized ray direction.
        volume: 3D array of scalar values.
        isovalue: Threshold T for isosurface.
        octree: Root node of the min-max octree.
        
    Returns:
        Color of the pixel.
    """
    # Initialize ray parameters
    t_min = 0.0
    t_max = volume.max_extent_distance(ray_origin, ray_direction)
    
    # Traverse octree to find relevant leaf nodes or exit volume
    current_node = octree
    relevant_nodes = []
    
    # Stack for traversal
    stack = [current_node]
    
    while stack:
        node = stack.pop()
        
        # Empty space skipping: if isovalue is not in [min, max], skip subtree
        if isovalue < node.min_val or isovalue > node.max_val:
            continue
            
        # If leaf node, add to list for marching
        if node.is_leaf:
            relevant_nodes.append(node)
        else:
            # Push children to stack (order may vary)
            for child in node.children:
                stack.append(child)
                
    # March through relevant regions
    t = t_min
    color = background_color
    
    while t < t_max:
        # Sample volume at current ray position
        pos = ray_origin + ray_direction * t
        val = sample_volume(volume, pos)
        
        # Check for isosurface crossing
        if val >= isovalue:
            # Refine intersection using binary search
            t_hit = binary_search_intersection(ray_origin, ray_direction, t, t_max, volume, isovalue)
            
            # Compute normal at intersection
            normal = compute_normal(volume, pos_at_t(ray_origin, ray_direction, t_hit))
            
            # Apply Gooch shading
            color = apply_gooch_shading(normal)
            break
        else:
            # Step forward
            t += 0.5 * cell_edge_length
            
    return color

def binary_search_intersection(ray_origin, ray_direction, t_start, t_end, volume, isovalue):
    """
    Perform 3 iterations of binary search to find isosurface intersection.
    """
    for _ in range(3):
        t_mid = (t_start + t_end) / 2.0
        pos_mid = ray_origin + ray_direction * t_mid
        val_mid = sample_volume(volume, pos_mid)
        
        # Determine which side of the threshold we are on relative to start
        # Assuming start is outside (val < isovalue) and end is inside (val >= isovalue)
        # Or vice versa. This logic assumes a sign change between t_start and t_end.
        # For robustness, check signs.
        val_start = sample_volume(volume, ray_origin + ray_direction * t_start)
        
        if (val_start < isovalue) == (val_mid < isovalue):
            t_start = t_mid
        else:
            t_end = t_mid
            
    return (t_start + t_end) / 2.0

def compute_normal(volume, pos):
    """
    Compute normal using central differences.
    """
    eps = 1e-5
    dx = sample_volume(volume, pos + vec3(eps, 0, 0)) - sample_volume(volume, pos - vec3(eps, 0, 0))
    dy = sample_volume(volume, pos + vec3(0, eps, 0)) - sample_volume(volume, pos - vec3(0, eps, 0))
    dz = sample_volume(volume, pos + vec3(0, 0, eps)) - sample_volume(volume, pos - vec3(0, 0, eps))
    return normalize(vec3(dx, dy, dz))
```

**Parameters:**
*   **Binary Search Iterations:** 3 [S1][S3, unofficial].
*   **Ray Step Size:** $0.5 \times \text{cell edge length}$ [S1][S3, unofficial].
*   **Min-Max Cells:** In GPU rasterization variants, $32^3$ min/max cells resulted in the best performance [S6].

**Edge Cases and Pitfalls:**
*   **Cell Size Trade-off:** A large number of small cells induces a tight bound of the isosurface but comes with large overhead due to primitives. A small number of large cells reduces overhead but bounds the isosurface loosely [S6].
*   **Precision:** Binary search refinement assumes a sign change between the start and end of the search interval. Degenerate cases where the isosurface is tangent to the grid may require careful handling [G].
*   **Memory:** While auxiliary data is low (1-4 MB for depth $\ge$ 7), the volume data itself can be large. Lossless compression into an octree can reduce size to <25% of the original 3D array [S4, S5, unofficial].

## Subtopics

### [Object-Order Iso-Surface Rendering with Macro-Cells](1757-object-order-iso-surface-rendering-with-macro-cells.md)
An iso-surface rendering algorithm based on object-order ray casting that subdivides data into macro-cells to build a min/max octree, reducing traversal steps by rasterizing hexagonal footprints. It builds on [Ray-marching reference (min-max octree accelerated)](1554-ray-marching-reference-min-max-octree-accelerated.md) by replacing per-pixel octree traversal with macro-cell rasterization to reuse traversal paths for multiple pixels.

### [Interactive Large Iso-Surface Ray Tracing (DeMarle et al.)](12725-interactive-large-iso-surface-ray-tracing-demarle-et-al.md)
An out-of-core iso-surface ray tracing engine designed to render volume datasets exceeding single-machine main memory limits by leveraging a PC cluster. It employs a hierarchical grid acceleration structure with a three-level bricking approach to minimize memory overhead, enabling interactive frame rates for terabyte-scale data. It differs from its parent [Ray-marching reference (min-max octree accelerated)](1554-ray-marching-reference-min-max-octree-accelerated.md) by using a rectilinear hierarchical grid instead of an octree, and from [Object-Order Iso-Surface Rendering with Macro-Cells](1757-object-order-iso-surface-rendering-with-macro-cells.md) by relying on image-order ray traversal and distributed out-of-core data loading rather than object-order macro-cell processing.

## Operating characteristics
*   **Time Complexity:** Depends on the sparsity of the volume and the depth of the octree. Empty space skipping significantly reduces the number of samples taken.
*   **Memory Complexity:** Auxiliary octree data requires 1–4 MB for depth $\ge$ 7 [S3, unofficial][S2]. The volume data can be compressed into 10–30% of its original size using the octree structure [S5, unofficial].
*   **Performance:**
    *   On an Intel Core Duo 2.16 GHz laptop with 2 GB RAM, frame rates were 2.4, 1.3, and 3.3 fps for LLNL Richtmyer-Meshkov, Utah CSAFE heptane, and a closer view, respectively [S4, unofficial].
    *   On a 16-node NUMA 2.4 GHz Opteron workstation, frame rates were 17.9, 9.8, and 22.0 fps for the same scenes [S4, unofficial].
    *   Performance gain up to $\approx 4.95$-fold using fine-grained min/max cells for efficient empty space skipping [S6].
    *   Outperformed by raster occlusion culling at high resolutions (e.g., 3840×2160), but remains more efficient at low-to-medium resolutions for sparse datasets like Chameleon [S1].
*   **Strengths:**
    *   Applicable to a wide range of volumes [S2][S3, unofficial].
    *   Low auxiliary memory overhead [S3, unofficial][S2].
    *   Guarantees continuous isosurfaces [S4, unofficial].
    *   Allows rendering of extremely large and time-variant data on multicore architectures [S5, unofficial].
*   **Limitations:**
    *   Performance is underwhelming compared to GPU volume renderers [S5, unofficial].
    *   Dependent on rendering resolution in some implementations (fragment shader based) [S3, unofficial][S6].
    *   Sphere Tracing and Segment Tracing are outperformed by octrees with depth $\ge$ 7 [S2][S3, unofficial].

## Background
The method originates from work by Knoll et al. [KWPH06], who combined the min/max acceleration functions of Wilhelms and Van Gelder [WV92] with a compressed format similar to the "cells octree" of Velasco and Torres [VT01]. Unlike Velasco and Torres, who built the octree around cells, Knoll et al. built it directly around voxels [S5, unofficial]. This required a fast neighbor-finding scheme (borrowed and improved from Frisken and Perry [FP02]) to retrieve cell corner values during ray tracing [S5, unofficial]. The traversal is similar to that proposed by Gargantini and Atkinson [GA93] [S5, unofficial]. It is used as a baseline reference for direct isosurface visualization performance in evaluations [S3, unofficial].

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://doi.org/10.1111/cgf.14670, fetched 2026-10-27
[S2] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
[S3] Efficient Direct Isosurface Rasterization of Scalar Volumes - EG, https://diglib.eg.org/items/8a65e0f0-c048-4fce-9d0b-5acf07452485, fetched 2026-10-30 (unofficial)
[S4] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.researchgate.net/publication/261082346_Interactive_Isosurface_Ray_Tracing_of_Large_Octree_Volumes, fetched 2026-10-29 (unofficial)
[S5] PDF A Survey of Octree Volume Rendering Methods, https://www.sci.utah.edu/~knolla/octsurvey.pdf, fetched 2026-10-29 (unofficial)
[S6] PDF GPU Isosurface Raycasting of FCC Datasets, https://minho-kim-uos.github.io/papers/kim12gpu.pdf, fetched 2026-10-29
