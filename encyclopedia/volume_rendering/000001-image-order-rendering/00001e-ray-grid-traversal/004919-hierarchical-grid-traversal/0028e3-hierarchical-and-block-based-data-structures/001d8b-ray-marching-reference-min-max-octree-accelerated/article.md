# Ray-marching reference (min-max octree accelerated)

Also called: Min-Max Octree Ray Casting, Min-Max Octree Ray-Marching, Min-Max Octree Ray-Marching Reference, Knoll et al. octree ray tracing, Interactive Isosurface Ray Tracing (Knoll et al.)

## Overview
This method is an accelerated direct isosurface visualization technique that uses a min-max octree to facilitate empty space skipping during ray marching [S1, S2][S3, unofficial]. It serves as a performance baseline for general volume rendering, offering negligible memory overhead for the input volume and low auxiliary data requirements (1–4 MB for octree depth $\ge$ 7) [S3, unofficial]. Unlike Sphere Tracing, which requires signed distance functions, or Segment Tracing, which is limited to hierarchical skeletal implicit surfaces, this approach is applicable to a wide range of scalar volumes [S2][S3, unofficial]. It guarantees continuous isosurfaces that locally interpolate the input data, avoiding the topological artifacts of mesh extraction techniques like marching cubes [S4, unofficial].

## How it works
The method constructs a hierarchical acceleration structure (min-max octree) from the scalar volume data and performs ray marching through this structure to find isosurface intersections.

### Inputs and Outputs
*   **Input:** A 3D scalar volume dataset $V$ of dimensions $D_x \times D_y \times D_z$. The data is typically padded to the nearest power of two along all three cardinal axes to facilitate straightforward octree construction [S1][S3, unofficial].
*   **Output:** A rendered image of the isosurface defined by a threshold value $T$.

### Data Structures
**Min-Max Octree:**
The octree is built directly around voxels (as opposed to cells) [S5, unofficial]. Each node in the octree stores aggregate information derived from its children:
*   **Internal Nodes:** Store the minimum ($min$) and maximum ($max$) scalar values of all voxels in their subtree. These values are used to determine if the isosurface threshold $T$ lies within the node's range. If $T < min$ or $T > max$, the node is empty with respect to the isosurface and can be skipped [S1, S2][S3, unofficial][S6].
*   **Leaf Nodes:** Store the actual scalar voxel values.
*   **Construction:** The octree is constructed bottom-up. A common implementation involves creating a MIP texture from a min-max texture used to determine block occupancy [S1][S3, unofficial]. The base level of the octree has equal size along all three cardinal axes [S1][S3, unofficial].

**Voxel Data:**
The scalar values are stored in a format accessible during traversal. In some implementations, the acceleration tree and scalar data are embedded in a single structure [S4, unofficial]. For GPU implementations, the volume is often stored as a 3D texture [S6].

### Algorithm Steps

1.  **Octree Construction:**
    *   Pad volume dimensions to powers of two if necessary [S1][S3, unofficial].
    *   Compute min/max values for each voxel or block.
    *   Build the octree hierarchy by aggregating min/max values from children to parents [S1][S3, unofficial].
    *   *Note:* For CPU-based implementations on large datasets, lossless compression into an octree representation can reduce memory footprint to 10–30% of the original size [S5, unofficial]. Optimized hash schemes may be used for point location and neighbor-finding [S4, unofficial].

2.  **Ray Initialization:**
    *   For each pixel, generate a ray origin $O$ and direction $D$ based on the camera parameters.

3.  **Ray Traversal (Empty Space Skipping):**
    *   Traverse the min-max octree from the root.
    *   At each node, check if the isosurface threshold $T$ is within the node's $[min, max]$ range.
    *   If $T \notin [min, max]$, prune the subtree (skip the node) [S1, S2][S3, unofficial].
    *   If $T \in [min, max]$, descend into the children.
    *   *GPU Implementation Detail:* To compute entry and exit points efficiently, cubes corresponding to one level of the min/max octree can be rendered. All vertices of a cube have the same 2D texture coordinates containing the min/max values. Exit positions are computed by rendering cubes with `glCullFace(GL FRONT)` and `glDepthFunc(GL LESS)`. Entry positions are computed by rendering cubes again with `glCullFace(GL BACK)` and `glDepthFunc(GL GREATER)`. A fragment shader discards fragments if the cube is empty or does not contain the isosurface level [S6].

4.  **Ray Marching:**
    *   Once a leaf node containing the isosurface is identified, march along the ray within that leaf.
    *   **Step Size:** Use a step size of $0.5 \times$ cell edge length [S1][S3, unofficial].
    *   Sample the scalar value $S$ at the current ray position.
    *   Check for isosurface intersection: if the sign of $(S - T)$ changes between the previous and current sample, an intersection exists within the step.

5.  **Isosurface Refinement:**
    *   Refine the intersection position using exactly three iterations of a binary search algorithm [S1][S3, unofficial].
    *   *Pseudocode for Refinement:*
        ```python
        def refine_intersection(ray_origin, ray_direction, start_pos, end_pos, threshold, iterations=3):
            pos_a = start_pos
            val_a = sample_volume(ray_origin + ray_direction * distance_to(pos_a, ray_origin))
            pos_b = end_pos
            val_b = sample_volume(ray_origin + ray_direction * distance_to(pos_b, ray_origin))
            
            for _ in range(iterations):
                mid_pos = (pos_a + pos_b) / 2.0
                dist_mid = distance_to(mid_pos, ray_origin)
                val_mid = sample_volume(ray_origin + ray_direction * dist_mid)
                
                if (val_mid - threshold) * (val_a - threshold) < 0:
                    pos_b = mid_pos
                    val_b = val_mid
                else:
                    pos_a = mid_pos
                    val_a = val_mid
            return pos_b
        ```

6.  **Normal Computation and Shading:**
    *   Compute surface normals on the fly using the central difference method upon finding an isosurface intersection [S1][S3, unofficial].
    *   *Pseudocode for Normal:*
        ```python
        def compute_normal(pos, volume):
            epsilon = 0.5 # or cell edge length
            normal = vec3(
                sample_volume(pos + vec3(epsilon, 0, 0)) - sample_volume(pos - vec3(epsilon, 0, 0)),
                sample_volume(pos + vec3(0, epsilon, 0)) - sample_volume(pos - vec3(0, epsilon, 0)),
                sample_volume(pos + vec3(0, 0, epsilon)) - sample_volume(pos - vec3(0, 0, epsilon))
            )
            return normalize(normal)
        ```
    *   Apply Gooch shading using the computed normals [S1][S3, unofficial].

### Parameters
*   **Binary Search Iterations:** 3 [S1][S3, unofficial].
*   **Ray Step Size:** $0.5 \times$ cell edge length [S1][S3, unofficial].
*   **Octree Depth:** Depth $\ge$ 7 is recommended for optimal performance and low memory overhead [S2][S3, unofficial].
*   **Min-Max Cells:** In GPU implementations using cube rendering for entry/exit, a number of $32^3$ min-max cells resulted in the best performance in experiments [S6].

### Edge Cases and Pitfalls
*   **Cell Size Trade-off:** A large number of small cells induces a tight bound of the isosurface but comes with large overhead due to primitives. A small number of large cells reduces overhead but bounds the isosurface loosely [S6].
*   **Precision:** The binary search refinement assumes sufficient precision in volume sampling.
*   **Boundaries:** Ensure ray origins and directions are correctly transformed into volume space.

## Subtopics

### [Object-Order Iso-Surface Rendering with Macro-Cells](1757-object-order-iso-surface-rendering-with-macro-cells.md)
An iso-surface rendering algorithm based on object-order ray casting that subdivides data into macro-cells to build a min/max octree, reducing traversal steps by rasterizing hexagonal footprints. It builds on this method by replacing per-pixel octree traversal with macro-cell rasterization to reuse traversal paths for multiple pixels.

### [Interactive Large Iso-Surface Ray Tracing (DeMarle et al.)](12725-interactive-large-iso-surface-ray-tracing-demarle-et-al.md)
An out-of-core iso-surface ray tracing engine designed to render volume datasets exceeding single-machine main memory limits by leveraging a PC cluster. It employs a hierarchical grid acceleration structure with a three-level bricking approach to minimize memory overhead, enabling interactive frame rates for terabyte-scale data. It differs from this method by using a rectilinear hierarchical grid instead of an octree, and from [Object-Order Iso-Surface Rendering with Macro-Cells](1757-object-order-iso-surface-rendering-with-macro-cells.md) by relying on image-order ray traversal and distributed out-of-core data loading rather than object-order macro-cell processing.

## Operating characteristics
*   **Time Complexity:** Depends on the sparsity of the volume and the octree depth. Empty space skipping significantly reduces the number of samples required.
*   **Memory Complexity:** Auxiliary octree data memory requirement is 1 to 4 megabytes for octrees with a depth of 7 or more [S3, unofficial]. The input volume memory overhead is considered negligible [S3, unofficial]. Lossless compression into an octree volume can occupy less than one quarter the size of the original 3D array [S4, unofficial].
*   **Measured Performance:**
    *   On an Intel Core Duo 2.16 GHz laptop with 2 GB RAM, frame rates were 2.4, 1.3, and 3.3 fps for three test scenes (LLNL Richtmyer-Meshkov, Utah CSAFE heptane, and a closer view) [S4, unofficial].
    *   On a 16-node NUMA 2.4 GHz Opteron workstation, frame rates were 17.9, 9.8, and 22.0 fps for the same three test scenes [S4, unofficial].
    *   On an NVIDIA RTX 3090, the method serves as a baseline, with more advanced rasterization approaches outperforming it at high resolutions (e.g., 3840×2160), but the ray-marching approach remains more efficient at low-to-medium resolutions for sparse datasets like Chameleon [S1].
    *   Performance gain up to $\approx 4.95$-fold using fine-grained min/max cells for efficient empty space skipping [S6].
*   **Strengths:**
    *   Applicable to a wide range of volumes [S2][S3, unofficial].
    *   Low auxiliary memory overhead [S3, unofficial].
    *   Guarantees continuous isosurfaces [S4, unofficial].
    *   Allows rendering of extremely large and time-variant data on multicore architectures [S5, unofficial].
*   **Limitations:**
    *   Performance is underwhelming compared to GPU volume renderers [S5, unofficial].
    *   Outperformed by raster occlusion culling approaches at high resolutions [S1].
    *   Dependent on rendering resolution in some implementations (e.g., fragment shader-based) [S3, unofficial].

## Background
The method originates from the work of Knoll et al., who combined the min/max acceleration functions of Wilhelms and Van Gelder [WV92] with a compressed format similar to the "cells octree" of Velasco and Torres [VT01] [S5, unofficial]. Knoll et al. built the octree directly around voxels, requiring a fast neighbor-finding scheme (borrowed and improved from Frisken and Perry [FP02]) to retrieve cell corner values during ray tracing [S5, unofficial]. Ray traversal is performed on the min/max octree structure using a traversal similar to that proposed by Gargantini and Atkinson [GA93] [S5, unofficial]. The method is used as a baseline reference for direct isosurface visualization performance in evaluations [S3, unofficial].

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://doi.org/10.1111/cgf.14670, fetched 2026-10-27
[S2] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
[S3] Efficient Direct Isosurface Rasterization of Scalar Volumes - EG, https://diglib.eg.org/items/8a65e0f0-c048-4fce-9d0b-5acf07452485, fetched 2026-10-30 (unofficial)
[S4] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.researchgate.net/publication/261082346_Interactive_Isosurface_Ray_Tracing_of_Large_Octree_Volumes, fetched 2026-10-29 (unofficial)
[S5] PDF A Survey of Octree Volume Rendering Methods, https://www.sci.utah.edu/~knolla/octsurvey.pdf, fetched 2026-10-29 (unofficial)
[S6] PDF GPU Isosurface Raycasting of FCC Datasets, https://minho-kim-uos.github.io/papers/kim12gpu.pdf, fetched 2026-10-29
