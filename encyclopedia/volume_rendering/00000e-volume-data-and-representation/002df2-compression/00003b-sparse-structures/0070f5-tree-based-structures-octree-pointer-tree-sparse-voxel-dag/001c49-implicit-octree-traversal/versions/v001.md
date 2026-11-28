# Implicit Octree Traversal

Also called: ImplicitOctree

## Overview
Implicit Octree Traversal is a method for constructing sparse octrees from distance fields to facilitate isosurface extraction, specifically targeting dual contouring or marching cubes on sparse data [S1, unofficial]. It differs from explicit octree structures by using Morton coordinates to implicitly define the tree hierarchy, allowing for efficient point location and neighbor retrieval without storing explicit parent pointers or full node metadata [S1, unofficial]. Unlike [Implicit Octree Storage](939-implicit-octree-storage.md), which focuses on the storage layout, this method defines the traversal algorithm to build the primal vertex map and locate dual vertices for mesh generation [S1, unofficial].

## How it works
**Inputs:** A distance field function (scalar or directed), maximum octree depth $N$.
**Outputs:** A sparse octree structure containing distance values at leaf nodes, and a map of primal vertices with their associated levels.

**Data Structures:**
1.  **ImplicitOctree**: Contains `max_depth` (usize). Internally uses a `LinearHashedOctree` to store distance values at Morton keys.
2.  **primal_vertices**: A `HashMap<Morton, usize>` mapping Morton coordinates of primal vertices to their tree level.

**Parameters:**
*   `max_depth` ($N$): The maximum depth of the octree. The effective grid resolution is $2^N \times 2^N \times 2^N$ [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization**:
    Create the `ImplicitOctree` instance with `max_depth = N`. This defines the bounding box and resolution limit [S1, unofficial].

2.  **Collect Primal Vertices**:
    Iterate through all leaf nodes of the octree. For each leaf key, generate the 8 primal vertex Morton coordinates corresponding to the corners of the leaf voxel.
    *   For each corner $i \in [0, 7]$:
        *   Compute `vertex` = `key.primal_vertex(level, i)`.
        *   If `vertex` is not the null Morton key:
            *   Check if `vertex` exists in `primal_vertices`.
            *   If it exists and the current `level` is greater than the stored `existing_level`, update the entry: `primal_vertices[vertex] = level`.
            *   Otherwise, insert the entry: `primal_vertices[vertex] = level` [S1, unofficial].

3.  **Traverse Primal Vertices**:
    For each `(key, level)` pair in `primal_vertices`:
    *   For each corner index $i \in [0, 7]$:
        *   Compute the target dual vertex Morton coordinate: `m` = `key.dual_vertex(level, REMAP_CUBE[i])`.
        *   While `m` is not the null Morton key:
            *   Query the octree for node `m`.
            *   If the node exists and returns a `distance` value:
                *   Set `keys[i] = m`.
                *   Set `corners[i] = m.center()`.
                *   Set `values[i] = distance`.
                *   Break the while loop.
            *   If the node does not exist:
                *   Set `m = m.parent()`.
    *   *Note*: If the loop terminates with `m` being null, the corner data is incomplete. The source code indicates this state requires handling, potentially by falling back to parent nodes or marking the cell as invalid [S1, unofficial].

**Formulas:**
*   **Crossing Point Calculation** (for edge interpolation):
    Given two endpoints $a, b$ with distances $d_a, d_b$ and positions $p_a, p_b$:
    $$ \delta = d_b - d_a $$
    $$ t = \begin{cases} 0.5 & \text{if } \delta = 0.0 \\ -\frac{d_a}{\delta} & \text{otherwise} \end{cases} $$
    $$ p_{cross} = p_a (1 - t) + p_b t $$
    For directed distance fields, the calculation is axis-aligned:
    $$ \text{axis} = \text{argmax}_k |p_a[k] - p_b[k]| $$
    $$ \delta = d_b[\text{axis}] - d_a[\text{axis}] $$
    $$ t = \begin{cases} 0.5 & \text{if } \delta = 0.0 \\ -\frac{d_a[\text{axis}]}{\delta} & \text{otherwise} \end{cases} $$
    $$ p_{cross} = p_a (1 - t) + p_b t $$ [S1, unofficial]

**Edge Cases and Pitfalls:**
*   **Directed Distance Fields**: The implementation notes that constructing an octree over a directed distance field is not fully implemented. The code includes a `TODO` comment indicating this limitation [S1, unofficial].
*   **Missing Nodes**: If a dual vertex is not found in the octree and its parent chain is exhausted without finding a value, the corner data remains unset. This may lead to degenerate cells during mesh extraction [S1, unofficial].
*   **Precision**: The use of floating-point distances and Morton coordinates requires careful handling of equality checks and boundary conditions, particularly when `delta` is zero [S1, unofficial].

## Subtopics

## Operating characteristics
*   **Time Complexity**: Point location is $O(\log N)$ in the depth of the tree, where $N$ is the maximum depth [S1, unofficial]. The collection of primal vertices requires iterating over all leaf nodes, which is proportional to the number of occupied voxels.
*   **Memory Complexity**: The memory usage is proportional to the number of leaf nodes in the sparse octree, plus the size of the `primal_vertices` hash map. The use of a hash map for primal vertices adds overhead but allows for efficient lookups [S1, unofficial].
*   **Strengths**: Efficient representation of sparse data; no need for explicit parent pointers; supports high-resolution details via deep octrees without storing empty space.
*   **Limitations**: Not fully implemented for directed distance fields; requires careful handling of missing nodes during traversal [S1, unofficial].

## Background
The method is derived from the `isosurface` crate by Tristam MacDonald, which implements dual contouring and marching cubes on sparse octrees [S1, unofficial]. The approach builds on the principles of implicit surface representation and sparse voxel octrees, aiming to provide a flexible and memory-efficient way to extract meshes from distance fields [S1, unofficial].

## Sources
[S1] https://github.com/swiftcoder/isosurface, https://github.com/swiftcoder/isosurface, fetched 2026-10-26 (unofficial)
