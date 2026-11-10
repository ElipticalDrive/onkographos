# Sparse Octree Filtering with Node Map

Also called: Light-view Map Node Map, Node Map Heuristic, Node Map MIP-map Filtering, Thread Termination for Sparse Filtering

## Overview
Sparse Octree Filtering with Node Map reduces the computational overhead of MIP-map generation for sparse voxel octrees by eliminating redundant thread execution during the filtering pass [S1, S2]. It achieves this by constructing a 2D hierarchical node map derived from the light-view map, which allows threads to detect when they are targeting the same octree node and terminate all but one [S1, S2]. This method sets itself apart from naive parallel filtering by explicitly handling the spatial coherence of photon distribution in the light-view map, yielding a speedup of greater than 2x compared to methods without this heuristic [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** The light-view map (a 2D texture where each pixel corresponds to a photon or light sample) and the sparse voxel octree structure [S1, S2].
*   **Output:** A filtered MIP-map pyramid for the octree, computed with reduced thread count [S1, S2].

**Data Structures:**
1.  **Light-View Map:** A 2D array where each pixel $(u, v)$ stores a photon index or position [S2].
2.  **Node Map:** A 2D MIP-map pyramid derived from the light-view map [S1, S2].
    *   **Level 0 (Lowest/Highest Resolution):** Each pixel $(u, v)$ stores the index of the 3D leaf node in the sparse voxel octree that contains the photon corresponding to the light-view map pixel at $(u, v)$ [S1, S2].
    *   **Level $L > 0$ (Higher/Lower Resolution):** Each pixel stores the index of the lowest common ancestor (LCA) node in the octree for all leaf nodes referenced by the pixels in the previous level ($L-1$) that map to this pixel [S1, S2].

**Algorithm Steps:**
1.  **Construct Node Map:**
    *   For the lowest level of the node map, iterate through all pixels of the light-view map. For each pixel, determine the leaf node index containing the associated photon and store it in the corresponding node map pixel [S1, S2].
    *   For each subsequent level $L$ of the node map, aggregate the leaf node indices from level $L-1$. For each pixel in level $L$, compute the lowest common ancestor of all non-empty leaf node indices from the corresponding region in level $L-1$. Store the LCA index in the node map pixel [S1, S2].
2.  **Filtering Pass (Parallel Execution):**
    *   Launch one thread per pixel of the lowest level of the node map for each MIP-mapping pass [S1, S2].
    *   Each thread $T$ corresponding to pixel $(u, v)$ at the current node map level performs the following:
        1.  Determine the target octree node for the filtering operation by looking up the node map at the current level [S1, S2].
        2.  **Redundancy Check:** Before computing the filtered value, the thread checks if another thread has already claimed or is computing the same target node by inspecting the node map at the current level to identify if the target node is a common ancestor for multiple threads [S1, S2].
        3.  **Termination Heuristic:** If a common ancestor is detected (i.e., multiple threads are targeting the same node), terminate all threads detecting this common ancestor except the one with the "upper left" position (lowest row index, then lowest column index) [S1, S2].
        4.  **Computation:** If the thread is not terminated, it computes the filtered MIP-map value for the target octree node using standard filtering logic [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sparse_octree_filtering_with_node_map(light_view_map, octree, node_map_levels):
    # Step 1: Construct Node Map Pyramid
    # Level 0: Map light-view pixels to leaf node indices
    level_0 = [[None] * light_view_map.width for _ in range(light_view_map.height)]
    for y in range(light_view_map.height):
        for x in range(light_view_map.width):
            photon = light_view_map[y, x]
            if photon is not None:
                level_0[y][x] = octree.find_leaf_node(photon)
    node_map_levels[0] = level_0

    # Higher levels: Map to Lowest Common Ancestor (LCA)
    for level in range(1, len(node_map_levels)):
        prev_level = node_map_levels[level - 1]
        curr_h = len(prev_level) // 2
        curr_w = len(prev_level[0]) // 2
        curr_level = [[None] * curr_w for _ in range(curr_h)]

        for y in range(curr_h):
            for x in range(curr_w):
                leaf_indices = []
                for dy in range(2):
                    for dx in range(2):
                        idx = prev_level[y * 2 + dy][x * 2 + dx]
                        if idx is not None:
                            leaf_indices.append(idx)

                if leaf_indices:
                    curr_level[y][x] = octree.compute_lca(leaf_indices)
                else:
                    curr_level[y][x] = None

        node_map_levels[level] = curr_level

    # Step 2: Parallel Filtering Pass
    # Launch one thread per pixel of the lowest node map level
    # (Thread launch mechanism abstracted; logic per thread follows)
    for y in range(len(node_map_levels[0])):
        for x in range(len(node_map_levels[0][0])):
            filter_thread(y, x, node_map_levels)

def filter_thread(y, x, node_map_levels):
    # Determine target node for this thread's work at the current MIP level
    # The node map at level L_node_map corresponds to the current MIP pass level.
    current_level_idx = 0 # Placeholder for current MIP pass level
    target_node = node_map_levels[current_level_idx][y][x]

    if target_node is None:
        return

    # Check for common ancestor redundancy
    # If multiple threads map to the same node, only the "upper left" one computes.
    # "Upper left" means smallest row index, then smallest column index.
    is_leader = True
    for other_y in range(y):
        for other_x in range(len(node_map_levels[current_level_idx][0])):
            if node_map_levels[current_level_idx][other_y][other_x] == target_node:
                is_leader = False
                break
        if not is_leader:
            break
    for other_x in range(x):
        if node_map_levels[current_level_idx][y][other_x] == target_node:
            is_leader = False
            break

    if not is_leader:
        return # Terminate thread

    # Compute the filtered value
    filtered_value = octree.compute_filtered_value(target_node)

    # Store result in the MIP-map
    mip_map[target_node] = filtered_value
```

**Parameters:**
*   **Node Map Resolution:** Matches the resolution of the light-view map at the lowest level [S1, S2].
*   **Thread Launch Count:** One thread per pixel of the lowest node map level [S1, S2].

**Edge Cases and Pitfalls:**
*   **Empty Pixels:** If a light-view map pixel contains no photon, the corresponding leaf node index is undefined and stored as null/sentinel [S1, S2].
*   **Boundary Conditions:** The "upper left" heuristic must be consistently defined (e.g., smallest row index, then smallest column index) to ensure exactly one thread computes the value [S1, S2].

## Subtopics

### [Anisotropic Voxel Filtering for Mipmap Generation](2996-anisotropic-voxel-filtering-for-mipmap-generation.md)
This method filters lighting data at the leaf nodes of an octree anisotropically along the positive and negative XYZ directions to prepare for cone tracing. It stores six directional values per voxel to preserve directional information.

### [Distribution over Levels (MIP-mapping)](3045-distribution-over-levels-mip-mapping.md)
Distribution over Levels propagates radiance values from leaf nodes to internal nodes of a sparse voxel octree to create a MIP-map pyramid for cone tracing. It distinguishes itself from standard isotropic downsampling by using a three-pass strategy to balance thread cost and explicitly handling shared boundary voxels in a $3 \times 3 \times 3$ brick structure.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The strategy yields a speedup of greater than 2x compared to not using the heuristic to terminate threads [S1, S2].
*   **Strengths:** Significantly reduces redundant computation in sparse octrees where many photons map to the same or nearby nodes [S1, S2].
*   **Limitations:** Requires the construction and maintenance of the node map. The "upper left" heuristic may not be optimal for all thread scheduling architectures [G].
*   **When to Use:** When filtering sparse voxel octrees for cone tracing, especially when the light-view map has high coherence (many photons in the same node) [G].
*   **When Not to Use:** If the octree is dense and the overhead of the node map construction outweighs the savings from thread termination [G].

## Background
This method was introduced in the GIVoxels algorithm [S2] for interactive indirect illumination using voxel cone tracing [S1, S2]. It addresses the inefficiency of parallel MIP-map generation for sparse voxel octrees, where many threads would otherwise perform the same work due to the spatial coherence of light samples [S1, S2].

## Sources
[S1] Interactive Indirect Illumination Using Voxel Cone Tracing, https://doi.org/10.1111/j.1467-8659.2011.02063.x, fetched 2026-10-26
[S2] PDF Interactive Indirect Illumination Using Voxel Cone Tracing, https://research.nvidia.com/sites/default/files/publications/GIVoxels-pg2011-authors.pdf, fetched 2026-10-27
