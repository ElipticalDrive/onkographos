# Sparse Octree Filtering with Node Map

Also called: Light-View 2D Node Map Generation, Light-view Map Node Map, Node Map Heuristic, Node Map MIP-map Filtering, Thread Termination for Sparse Filtering

## Overview
Sparse Octree Filtering with Node Map is a thread-synchronization and culling heuristic used during the MIP-mapping pass of sparse voxel octrees [S1, S3]. It solves the problem of redundant computation where multiple threads attempt to filter data into the same 3D octree node [S1]. By projecting the 3D octree structure onto a 2D hierarchy derived from the light-view map, the method identifies common ancestors among threads and terminates all but the "upper-left" thread, ensuring each node is filtered exactly once [S2, unofficial][S3]. It differs from [Software Z-buffer Brick Projection](22976-software-z-buffer-brick-projection.md) by using a hierarchical MIP-map structure for ancestor verification rather than a flat depth buffer, and it builds on [Fast Projection for Octree Nodes](2178-fast-projection-for-octree-nodes.md) by providing the synchronization mechanism for the subsequent filtering pass.

## How it works
**Inputs and Outputs:**
*   **Input:** A Light-View Map (LVM), which is a 2D MIP-map pyramid representing the projection of the sparse voxel octree [S1, S3].
*   **Input:** The current MIP-mapping pass level (from leaf to root) [S1].
*   **Output:** A 2D Node Map, which is a MIP-map pyramid of the same resolution hierarchy as the LVM [S1, S3].
*   **Output:** A set of active threads that perform the actual filtering, with redundant threads terminated [S3].

**Data Structures:**
The **Node Map** is a 2D array of integers, structured as a MIP-map pyramid [S1, S3].
*   **Level 0 (Lowest/Highest Resolution):** Each pixel $(u, v)$ stores the index of the 3D leaf node in the sparse voxel octree that contains the photon corresponding to the LVM pixel at $(u, v)$ [S1, S3].
*   **Level $k > 0$ (Higher/Lower Resolution):** Each pixel stores the index of the lowest common ancestor (LCA) node in the 3D octree for all leaf nodes mapped to the corresponding 4 pixels (or relevant block) in Level $k-1$ [S1, S3].

**Algorithm Steps:**
1.  **Construct Node Map:**
    *   For the lowest level of the Node Map, iterate through all pixels of the LVM.
    *   For each LVM pixel, identify the 3D leaf node containing the corresponding photon.
    *   Store the index of this 3D leaf node in the corresponding pixel of the Node Map Level 0 [S1, S3].
    *   For each subsequent level $k$ of the Node Map, compute the LCA of the 3D nodes stored in the corresponding block of pixels from Level $k-1$.
    *   Store this LCA index in the pixel at Level $k$ [S1, S3].

2.  **Launch Threads:**
    *   Launch one thread per pixel of the lowest level (Level 0) of the Node Map for the current MIP-mapping pass [S1][S2, unofficial][S3].

3.  **Thread Execution and Synchronization:**
    *   Each thread $T$ associated with pixel $(u, v)$ at the current Node Map level $L$ performs the following:
        *   Determine the target 3D octree node for the MIP-mapping computation. This is typically the node indicated by the Node Map at level $L$ for pixel $(u, v)$, or derived by traversing down from the LCA if the thread is responsible for a specific subset [S1, S3].
        *   Check the Node Map at level $L$ for the pixel $(u, v)$ to identify the node index $N_{curr}$ [S3].
        *   Compare $N_{curr}$ with the node indices of neighboring threads (specifically those that might map to the same 3D node) [S3].
        *   If a common ancestor $N_{common}$ is detected among a group of threads (i.e., multiple threads are targeting the same 3D node or its ancestor):
            *   Identify the thread with the smallest $(u, v)$ coordinates (e.g., upper-left-most) among those threads [S3].
            *   Terminate all other threads in the group [S2, unofficial][S3].
            *   Allow the surviving thread (upper-left) to compute the filtered values for the remaining contributions [S3].
        *   If no common ancestor conflict is detected, or if this thread is the designated survivor, proceed to compute the MIP-mapped value for the target 3D node [S1, S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sparse_octree_filtering_with_node_map(light_view_map, node_map, current_level):
    """
    light_view_map: 2D array of photons/intensity values, structured as MIP-map pyramid.
    node_map: 2D array of 3D node indices, structured as MIP-map pyramid.
    current_level: The level of the MIP-mapping pass being performed (0 = leaf, increasing towards root).
    """
    # Launch one thread per pixel at the lowest level of the node map
    # In a real GPU/CPU implementation, this is done via a parallel launch.
    # Here we simulate the logic for a single thread identified by (u, v) at level 0.
    
    # 1. Identify the target 3D node for this thread's responsibility
    # At the lowest level, the node map directly stores the leaf node index.
    # For higher levels, the thread might need to traverse or use the LCA.
    # The sources state threads look for the node to compute MIP-mapped value.
    
    # 2. Check for common ancestors using the Node Map
    # The Node Map at 'current_level' provides the LCA for the block of pixels.
    # Threads compare their current level's node map entry with neighbors.
    
    my_node_index = node_map[current_level][u][v]
    
    # Determine neighbors to check for conflicts.
    # The exact neighbor definition (e.g., 4-connected, 8-connected) is not specified,
    # but the goal is to find threads mapping to the same 3D node.
    neighbors = get_neighbors(u, v, node_map[current_level])
    
    is_survivor = True
    
    for neighbor in neighbors:
        neighbor_node_index = neighbor.node_index
        neighbor_coords = neighbor.coords
        
        # If we share the same node index, there is a conflict
        if neighbor_node_index == my_node_index:
            # Determine if this thread is the "upper-left" one
            # Upper-left is defined by smallest (u, v) coordinates
            if (neighbor_coords.u < u) or (neighbor_coords.u == u and neighbor_coords.v < v):
                is_survivor = False
                break
    
    # 3. Terminate or Compute
    if not is_survivor:
        return # Thread terminates
        
    # 4. Compute MIP-mapped value
    # The surviving thread computes the filtered value for the node.
    # This involves gathering contributions from photons in the light-view map
    # that map to this 3D node.
    filtered_value = compute_mip_mapped_value(light_view_map, my_node_index, current_level)
    
    # Store result (implementation detail depends on memory layout)
    store_result(my_node_index, filtered_value)
```

**Parameters:**
*   **Thread Termination Heuristic:** The "upper-left" rule is used to break ties. [S3]
*   **Speedup:** The strategy yields a speedup of $> 2\times$ compared to not using the heuristic [S1, S3].

**Edge Cases and Pitfalls:**
*   **Precision:** Not given in the sources.
*   **Boundaries:** Threads at the edge of the node map must handle neighbor checks carefully to avoid out-of-bounds access. [G]
*   **Degenerate Input:** If the light-view map is empty, no threads are launched. [G]

## Subtopics

### [Anisotropic Voxel Filtering for Mipmap Generation](2996-anisotropic-voxel-filtering-for-mipmap-generation.md)
This method filters lighting data at the leaf nodes of an octree anisotropically along the positive and negative XYZ directions to prepare for cone tracing. It stores six directional values per voxel to preserve directional information.

### [Distribution over Levels (MIP-mapping)](3045-distribution-over-levels-mip-mapping.md)
Distribution over Levels (MIP-mapping) filters radiance values from the lowest level of a sparse voxel octree and stores the results in higher levels to form a MIP-map hierarchy. It differs from regular grid MIP-mapping by explicitly handling the boundary sharing of 3x3x3 voxel bricks in an octree structure, requiring weighted averaging to account for overlapping voxels.

## Operating characteristics
*   **Time Complexity:** The construction of the Node Map is $O(N)$ where $N$ is the number of pixels in the LVM [S1]. The filtering pass is $O(P)$ where $P$ is the number of pixels, but with a reduction factor due to thread termination.
*   **Memory Complexity:** Requires storage for the Node Map, which is a MIP-map pyramid of the same resolution as the LVM [S1, S3].
*   **Measured Performance:** Yields $> 2\times$ speedup compared to not using the heuristic [S1, S3].
*   **Strengths:** Significantly reduces redundant computation in high-level nodes where many threads converge [S1].
*   **Limitations:** Requires the additional memory and computation to build and maintain the Node Map [S1].
*   **When to use:** When performing MIP-mapping on sparse voxel octrees with high thread concurrency [S1].
*   **When not to use:** If memory bandwidth is the primary bottleneck and the Node Map construction cost outweighs the filtering savings [G].

## Background
The method was introduced in the paper "Interactive Indirect Illumination Using Voxel Cone Tracing" by Bitterli et al. (2011) [S1]. It was developed to optimize the performance of voxel cone tracing by reducing the overhead of thread synchronization during the sparse octree MIP-mapping process [S1, S3]. The technique relies on the hierarchical structure of the light-view map to project 3D octree relationships into 2D [S1].

## Sources
[S1] Interactive Indirect Illumination Using Voxel Cone Tracing, https://doi.org/10.1111/j.1467-8659.2011.02063.x, fetched 2026-10-26
[S2] Implementations tasklist, https://github.com/wzchua/graphic/issues/1, fetched 2026-10-26 (unofficial)
[S3] PDF Interactive Indirect Illumination Using Voxel Cone Tracing, https://research.nvidia.com/sites/default/files/publications/GIVoxels-pg2011-authors.pdf, fetched 2026-10-27
