# Hierarchical Raster Occlusion Culling

Also called: Lee et al. hierarchical raster occlusion culling

## Overview
Hierarchical Raster Occlusion Culling is a fully GPU-driven technique that reduces rasterization overhead by performing visibility tests on a coarse hierarchy of group nodes rather than individual leaf objects, refining results via ray traversal to the leaf level [S1]. It distinguishes itself from standard Raster Occlusion Culling by replacing the exhaustive rasterization of all leaf proxies with a hierarchical culling pass, and from Iterative Raster Occlusion Culling by utilizing a hybrid raster-ray approach for full hierarchy descent [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A hierarchical spatial structure (e.g., octree) of voxel blocks; camera parameters.
*   **Output:** A list of visible leaf-level blocks for subsequent volume rendering.

**Data Structures:**
*   **Hierarchy Tree:** Nodes are organized in levels. Group nodes represent aggregated bounding volumes of their children. Leaf nodes represent individual voxel blocks.
*   **Proxy Geometry:** Each node (group or leaf) has an associated proxy geometry (typically an axis-aligned bounding box or simplified mesh) used for rasterization.

**Steps:**
1.  **Hierarchy Construction:** Build or traverse the spatial hierarchy. Assign proxy geometry to each node.
2.  **Coarse Rasterization:** Rasterize the proxy geometry of the group nodes at a higher level of the hierarchy. This determines which coarse regions potentially contain visible geometry.
3.  **Ray Traversal Refinement:** For regions identified as potentially visible in the coarse pass, perform ray traversal down the hierarchy tree. Rays are cast from the camera through the coarse visible regions to intersect with child nodes.
4.  **Leaf Visibility Determination:** Continue traversal until leaf nodes are reached. A leaf block is marked visible if the ray traversal confirms it is not occluded by previously processed geometry or if it passes the final rasterization test at the leaf level.
5.  **Stream Compaction:** Collect the indices of all visible leaf blocks into a dense list for rendering.

**Formulas:**
*   Visibility of a leaf node $L$ is determined by the intersection of rays $R$ with the hierarchy: $V(L) = \bigvee_{r \in R} (r \cap L \neq \emptyset \land \text{not occluded})$. [G] (Logical OR over rays; occlusion check against previously rendered geometry).

**Parameters:**
*   **Hierarchy Depth:** The number of levels to rasterize coarsely. Not given in the sources.
*   **Ray Count:** Number of rays per coarse region. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Ray-triangle intersection precision at leaf levels may cause flickering if not handled with epsilon offsets. [G]
*   **Degenerate Input:** Empty regions in the hierarchy should be skipped during traversal to avoid unnecessary ray casting. [G]

## Subtopics

### Iterative Raster Occlusion Culling
An iterative version of raster occlusion culling that works on a select number of hierarchy levels, applying spatial coherence exploitation through hierarchical grouping of objects [S1]. It differs from the fully GPU-driven hierarchical approach by potentially using a more iterative, possibly CPU-GPU hybrid or multi-pass rasterization strategy on specific levels rather than a single hybrid raster-ray descent [S1].

## Operating characteristics
*   **Time Complexity:** Reduced compared to full leaf rasterization, as only coarse nodes are rasterized initially. Complexity depends on hierarchy depth and ray traversal cost. Not given in the sources.
*   **Memory Complexity:** Requires storage for the hierarchy tree and proxy geometries. Not given in the sources.
*   **Strengths:** Significantly reduces rasterization overhead by avoiding per-leaf rasterization. Exploits spatial coherence.
*   **Limitations:** Ray traversal adds computational overhead. Performance may degrade in highly complex or unstructured scenes where hierarchy balancing is poor.
*   **When to use:** Large-scale volume rendering with high block counts where rasterization is the bottleneck.
*   **When not to use:** Scenes with very few blocks or where ray traversal overhead exceeds rasterization savings.

## Background
The concept of hierarchical grouping for occlusion culling was introduced by Martin et al. [MBW08]. Lee and Lee [LL20] proposed an iterative version of raster occlusion culling. Lee et al. [LJSL21] introduced the fully GPU-driven hierarchical raster occlusion culling algorithm, aiming to reduce overall rasterization overhead by combining coarse rasterization with ray traversal refinement [S1].

## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
