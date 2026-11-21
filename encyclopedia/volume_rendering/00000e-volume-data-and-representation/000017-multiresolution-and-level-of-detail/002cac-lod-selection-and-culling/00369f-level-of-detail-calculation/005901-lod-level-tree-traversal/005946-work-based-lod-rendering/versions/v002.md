# Work-based LOD Rendering

Also called: WorkTree4DRenderer

## Overview
Work-based LOD Rendering is a visualization technique that maps the computational effort required to locate a data leaf in a 4D sparse voxel octree to pixel color, rather than rendering the actual density or scalar values [S1, unofficial]. It solves the problem of visualizing traversal complexity and data sparsity distribution across a volume by treating the number of tree steps as the primary data channel. It differs from standard raycasting by discarding volumetric integration in favor of a binary search for the first occupied leaf, and from depth-based rendering by measuring steps in tree hierarchy levels rather than geometric distance [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 4D Sparse Voxel Octree (SVO) containing spatial $(x, y, z)$ and temporal $(t)$ dimensions; a camera configuration defining 3D rays; a specific time point $t$ for the 4D ray conversion; and a global maximum step counter (initialized to 0 or previous frame max).
- **Output:** A 2D texture array (RGBA) where each pixel stores a color derived from the step count required to find a data leaf for that pixel's ray.

**Data Structures:**
- **Tree:** A 4D Sparse Voxel Octree. Nodes have properties `isLeaf()` and `hasData()`.
- **Ray4D:** A ray structure extended with time. Created from a 3D ray and a scalar time point.
- **TreeTraverser:** An iterator object (`Tree4DTraverserDifferentSides`) that maintains the current node, step count, and termination state.
- **Texture Array:** A buffer of unsigned char values (RGBA), indexed by `partial_index_in_texture_array + x * 4`.

**Parameters:**
- `time_point`: The temporal coordinate for the 4D ray. Typical values depend on the dataset's time range $[min, max]$ [S1, unofficial].
- `max_step_count`: The global maximum step count observed across all pixels in the current frame, used for normalization [S1, unofficial].

**Algorithm:**
The method processes pixels in parallel using OpenMP. For each pixel, it performs a tree traversal to count steps until a leaf with data is found.

1.  Initialize an OpenMP parallel loop over pixel rows $y$:
    ```python
    for y in range(rc.n_y):
    ```
2.  For each pixel $(x, y)$:
    a.  Compute the 3D ray from the camera for pixel $(x, y)$.
    b.  Convert the 3D ray to a 4D ray using the current `time_point`:
        $$ \text{ray4D} = \text{convertRayTo4D}(\text{ray3D}, \text{time\_point}, 0.0) $$
    c.  Initialize the tree traverser:
        $$ \text{treeTraverser} = \text{Tree4DTraverserDifferentSides}(\text{tree}, \text{ray4D}) $$
    d.  Initialize `dataLeafFoundForThisPixel = false` and `stepcount = 0`.
    e.  Traverse the tree in a loop:
        ```python
        while not treeTraverser.isTerminated() and not dataLeafFoundForThisPixel:
            if treeTraverser.getCurrentNode().isLeaf() and treeTraverser.getCurrentNode().hasData():
                dataLeafFoundForThisPixel = true
            else:
                treeTraverser.step()
                stepcount += 1 # [G] Increment step count explicitly as implied by 'stepcount' usage
        ```
    f.  Update the global maximum step count:
        ```python
        if stepcount > max_step_count:
            max_step_count = stepcount
        ```
    g.  Map the `stepcount` to a color using a color map function:
        $$ \text{color} = \text{GetColour}(\text{stepcount}, 0.0, \text{max\_step\_count}) $$
    h.  Store the RGBA values in the texture array at index `partial_index_in_texture_array + x * 4`.

**Edge Cases and Pitfalls:**
- **No Data Leaf:** If the ray terminates without finding a leaf with data (`isTerminated()` becomes true), the `stepcount` reflects the depth of traversal until termination. The color map must handle this case, potentially mapping it to a background color or the maximum step count [S1, unofficial].
- **Precision:** The conversion from 3D to 4D ray assumes a linear time component. Precision errors in ray origin or direction may cause missed leaves in sparse regions [S1, unofficial].
- **Boundary Conditions:** The time point must be within the tree's temporal bounds $[min, max]$ to ensure valid 4D ray generation [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot D)$, where $N$ is the number of pixels and $D$ is the average depth of the tree traversal to find a data leaf or termination. In the worst case (empty space), $D$ is the height of the octree.
- **Memory Complexity:** $O(1)$ auxiliary memory per pixel (excluding the tree itself and the output texture). The algorithm is stateless between pixels except for the global `max_step_count`.
- **Performance:** Parallelized using OpenMP over pixel rows. Performance is limited by memory access patterns in the sparse voxel octree and the branching factor of the tree.
- **Strengths:** Provides a direct visualization of data sparsity and traversal cost. Useful for debugging tree construction and optimizing culling strategies.
- **Limitations:** Does not convey volumetric density or scalar values. Colors represent computational work, not physical properties.

## Background
Work-based LOD Rendering was implemented in the "4D-Sparse-Voxel-Octree-Renderer" project by joschout [S1, unofficial]. It is part of a suite of visualization techniques for 4D sparse voxel data, including depth rendering and time-point counting. The method leverages the inherent structure of the 4D SVO to provide insight into the efficiency of ray traversal algorithms.

## Sources
[S1] joschout/4D-Sparse-Voxel-Octree-Renderer, https://github.com/joschout/4D-Sparse-Voxel-Octree-Renderer, fetched 2026-10-26 (unofficial)
