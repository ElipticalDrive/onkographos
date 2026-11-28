# LOD Level Tree Traversal

Also called: LOD Raycasting with Stack Size Tracking, LODColor, LODColorTree4DRenderer, LODLevel, LODLevelTree4DRenderer, Optimized 4D Tree Rendering, Stack Depth LOD

## Overview
LOD Level Tree Traversal is a raycasting method for 4D sparse voxel octrees that determines the visual output of a pixel by terminating the tree traversal based on specific criteria (data presence, maximum depth, or projected size) and then mapping the traversal state to a color value [S1, unofficial]. It differs from standard volume rendering by decoupling the traversal termination logic from the final color computation, allowing the same traversal engine to render density, normals, or traversal complexity (stack depth) by swapping only the final color calculation step [S1, unofficial]. It builds on [Level of Detail Calculation](7836-level-of-detail-calculation.md) by using the calculated LOD limits to prune the search space.

## How it works
**Inputs:**
- A 4D sparse voxel octree (`tree`) with grid dimensions `gridsize_T` (time) and `gridsize_S` (spatial).
- A render context (`rc`) containing image dimensions `n_x`, `n_y`, camera parameters, and frustum width.
- A time point `t_pixel` for the 4D ray conversion.
- Parameters: `max_level` (maximum traversal depth), `pixel_size` (threshold for projected node size).

**Outputs:**
- A texture array (`texture_array`) of size `rc.n_y * rc.n_x * 4`, storing RGBA values per pixel.

**Data Structures:**
- `Tree4DTraverserDifferentSides`: Manages the traversal state, including the current node, stack depth, and projected size calculations.
- `Ray4D`: 4D ray representation (3D space + time).
- `MAP_COLOUR`: Structure containing `r`, `g`, `b` double components.

**Algorithm:**

1.  **Initialization:**
    - Determine the maximum possible stack depth:
      $$ size = \max(\text{tree} \to \text{gridsize\_T}, \text{tree} \to \text{gridsize\_S}) $$
      $$ \text{max\_stack\_size} = \lfloor \log_2(size) \rfloor + 1 $$
    - Initialize OpenMP threading to match the number of available processors:
      $$ \text{num\_threads} = \text{omp\_get\_num\_procs}() $$
    - For each pixel $(x, y)$ in the image:
      - Calculate the linear index in the texture array:
        $$ \text{partial\_index} = y \times (rc.n_y \times 4) $$
        $$ \text{index} = \text{partial\_index} + x \times 4 $$
      - Generate the 3D ray for the pixel:
        $$ \text{ray3D} = rc.\text{getRayForPixel\_rayparam}(x, y, t\_pixel) $$
      - Convert to 4D ray:
        $$ \text{ray4D} = \text{Ray4D::convertRayTo4D}(\text{ray3D}, t\_pixel, 0.0) $$
      - Initialize the traverser:
        $$ \text{treeTraverser} = \text{Tree4DTraverserDifferentSides}(\text{tree}, \text{ray4D}) $$
      - Initialize termination flags:
        $$ \text{dataLeafNodeHasBeenFound} = \text{false} $$
        $$ \text{nodeIsToSmall} = \text{false} $$
        $$ \text{reachedMaxLevelToRender} = \text{false} $$

2.  **Traversal Loop:**
    - While `!treeTraverser.isTerminated()` AND `!dataLeafNodeHasBeenFound` AND `!nodeIsToSmall` AND `!reachedMaxLevelToRender`:
      - **Check for Data Leaf:**
        - If `treeTraverser.getCurrentNode()->isLeaf()` AND `treeTraverser.getCurrentNode()->hasData()`:
          - Set `dataLeafNodeHasBeenFound = true`.
      - **Check Max Level:**
        - If `treeTraverser.stack_TraversalInfo_about_Node4Ds.size() >= max_level`:
          - Set `reachedMaxLevelToRender = true`.
      - **Check Projected Size:**
        - Calculate projected size of the current node relative to the pixel:
          $$ \text{projectedSizeOfCurrentNode} = \text{treeTraverser.getProjectedSizeOfCurrentNode}(t\_pixel) $$
        - If `projectedSizeOfCurrentNode <= pixel_size`:
          - Set `nodeIsToSmall = true`.
      - **Advance Traversal:**
        - If no termination condition is met, call `treeTraverser.step()`.

3.  **Color Calculation and Storage:**
    - If any termination condition is met (data leaf found, max level reached, or node too small):
      - Retrieve the current stack size (depth):
        $$ \text{current\_stack\_size} = \text{treeTraverser.stack\_TraversalInfo\_about\_Node4Ds.size()} $$
      - Calculate color based on the variant (see Subtopics). For the base LODColor variant:
        $$ \text{stack\_colour} = \text{GetColour}(\text{current\_stack\_size}, 0.0, \text{max\_stack\_size}) $$
        - Extract components: $R = \text{stack\_colour}.r$, $G = \text{stack\_colour}.g$, $B = \text{stack\_colour}.b$.
      - Store into texture array (scaled to [0, 255], Alpha = 1):
        $$ \text{texture\_array}[\text{index}] = \lfloor R \times 255.0 \rfloor $$
        $$ \text{texture\_array}[\text{index} + 1] = \lfloor G \times 255.0 \rfloor $$
        $$ \text{texture\_array}[\text{index} + 2] = \lfloor B \times 255.0 \rfloor $$
        $$ \text{texture\_array}[\text{index} + 3] = 1 $$

**Parameters:**
- `pixel_size`: Threshold for node projected size. Calculated as `pixel_radius * pixel_radius`, where `pixel_radius = 0.5 * (frustum_width / rc.n_x)` [S1, unofficial].
- `max_level`: Static member variable defining the maximum depth of the tree to traverse [S1, unofficial].
- `max_stack_size`: Calculated as $\lfloor \log_2(\max(\text{gridsize\_T}, \text{gridsize\_S})) \rfloor + 1$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** The projected size calculation must handle floating-point inaccuracies near the `pixel_size` threshold.
- **Boundary Conditions:** If the ray does not intersect the tree, `treeTraverser.isTerminated()` becomes true immediately, and the color calculation uses stack size 0.
- **Degenerate Input:** If `gridsize_T` or `gridsize_S` is 0, `max_stack_size` calculation may be undefined; sources do not specify handling for empty trees [S1, unofficial].

## Subtopics

### [LOD Tree Search](16352-lod-tree-search.md)
LOD Tree Search is a traversal algorithm for multidimensional (multikey) data structures, specifically 3D binary trees used in sparse voxel octrees, to efficiently locate leaf nodes containing an isosurface at a specific Level of Detail (LOD). It solves the problem of finding all cells where the scalar field value crosses a threshold without examining every node in the hierarchy. It is distinguished from sibling methods like [Work-based LOD Rendering](16373-work-based-lod-rendering.md) by its deterministic, key-based pruning strategy rather than workload balancing.

### [Work-based LOD Rendering](16373-work-based-lod-rendering.md)
Work-based LOD Rendering is a visualization technique that maps the computational effort required to locate a data-bearing leaf node in a 4D sparse voxel octree to pixel color, rather than rendering the underlying density or scalar values. It serves as a diagnostic or analytical tool to visualize traversal complexity and data sparsity distribution across the volume. Unlike standard raycasting which outputs physical properties, this method outputs the number of traversal steps, highlighting regions where the tree structure is deep or data is sparse.

### LODNormalTree4DRenderer
A renderer that traverses a 4D sparse voxel octree to find leaf nodes with data, then stores the normal vector of that leaf node as RGB color values in a texture array [S1, unofficial]. It differs from the base LODColor variant by extracting the normal vector from the leaf node data, normalizing it, and mapping the components from $[-1, 1]$ to $[0, 255]$ for storage [S1, unofficial]. The color calculation step is replaced with:
1. Retrieve normal: $\text{normal} = \text{tree} \to \text{data\_ptrs}[\text{treeTraverser.getCurrentNode()->data}] \to \text{normal}$.
2. Normalize: $\text{normal} = \text{normalize}(\text{normal})$.
3. Map to RGB:
   $$ R = 255 \times \frac{\text{normal}[0] + 1.0}{2.0} $$
   $$ G = 255 \times \frac{\text{normal}[1] + 1.0}{2.0} $$
   $$ B = 255 \times \frac{\text{normal}[2] + 1.0}{2.0} $$
4. Store $R, G, B$ and $\alpha=1$ into the texture array [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of pixels and the average depth of the tree traversal. The traversal is pruned by `max_level` and `pixel_size`, reducing the number of nodes visited per ray.
- **Memory Complexity:** Dominated by the storage of the 4D sparse voxel octree and the output texture array ($O(N \times M \times 4)$ for image dimensions $N, M$). The traversal stack depth is bounded by $\log_2(\max(\text{gridsize\_T}, \text{gridsize\_S}))$.
- **Performance:** Parallelized using OpenMP across pixels. Performance scales with the number of CPU cores.
- **Strengths:** Flexible output (color, normal, stack depth) by swapping the final calculation step. Efficient traversal using projected size culling.
- **Limitations:** Requires a 4D sparse voxel octree. The `pixel_size` threshold must be tuned to avoid over- or under-sampling.
- **When to use:** When visualizing traversal complexity, normals, or when the specific LOD level reached is more important than the density value itself.
- **When not to use:** When standard density-based volume rendering is required without modification.

## Background
The method originates from the 4D Sparse Voxel Octree Renderer project by joschout [S1, unofficial]. It is part of a family of renderers that traverse 4D trees to visualize different aspects of the data structure, including color, normals, and stack depth. The approach is used in research contexts for analyzing sparse voxel octree traversal efficiency and data distribution.

## Sources
[S1] joschout/4D-Sparse-Voxel-Octree-Renderer, https://github.com/joschout/4D-Sparse-Voxel-Octree-Renderer, fetched 2026-10-26 (unofficial)
