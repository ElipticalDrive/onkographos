# Sparse Octree Filtering with Node Map

Also called: Light-view Map Node Map, Node Map Heuristic, Node Map MIP-map Filtering, Thread Termination for Sparse Filtering

## Overview
Sparse Octree Filtering with Node Map is a thread-scheduling heuristic for MIP-map generation in sparse voxel octrees that reduces redundant computation by terminating duplicate threads [S1, S2]. It solves the problem of multiple threads performing identical filtering work on the same octree node, which is frequent at higher MIP-map levels [S1, S2]. It differs from standard parallel MIP-map generation by using a 2D auxiliary structure, the node map, to detect common ancestors among threads before computation begins [S1, S2].

## How it works
The method optimizes the parallel generation of MIP-map levels for a sparse voxel octree by preventing redundant thread execution. It relies on a 2D node map derived from the light-view map to identify which threads are targeting the same 3D octree node.

**Inputs and Outputs**
*   **Input:** The sparse voxel octree structure, the light-view map (a 2D array of photon indices), and the target MIP-map level for the current pass.
*   **Output:** Updated MIP-map level data for the octree nodes, with redundant threads terminated.

**Data Structures**
1.  **Light-View Map:** A 2D array where each pixel stores the index of a 3D leaf node in the sparse voxel octree that contains the corresponding photon [S1, S2].
2.  **Node Map:** A 2D MIP-map pyramid derived from the light-view map [S1, S2].
    *   **Lowest Level:** Pixels store the indices of the 3D leaf nodes containing the corresponding photons, identical to the light-view map [S1, S2].
    *   **Higher Levels:** Each pixel stores the index of the lowest common ancestor (LCA) node in the octree for the set of nodes referenced by the corresponding pixels in the previous (lower) level [S1, S2].

**Algorithm Steps**
1.  **Launch Threads:** Launch one thread per pixel of the lowest level of the node map for the current MIP-mapping pass [S1, S2].
2.  **Node Lookup:** For each thread, determine the target octree node for the current MIP-map level. This involves traversing the octree structure from the leaf node indicated in the light-view map up to the required level [S1, S2].
3.  **Common Ancestor Detection:** Before computing the filtered value, the thread checks the current level of the node map to see if another thread has already detected or is computing the same common ancestor node [S1, S2].
    *   This check verifies if the node index stored in the node map at the current thread's position (or a related position in the hierarchy) matches the target node of another active thread [S1, S2].
4.  **Termination Heuristic:** If a common ancestor is detected among multiple threads:
    *   Terminate all threads detecting this common ancestor, except for the thread corresponding to the "upper left" pixel in the node map [S1, S2].
    *   The surviving "upper left" thread is responsible for computing the filtered values for the remaining threads [S1, S2].
5.  **Computation:** The surviving threads compute the MIP-mapped values for their respective octree nodes [S1, S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume node_map is a 2D array of octree node indices, structured as a MIP-map pyramid.
# Assume octree is the sparse voxel octree structure.
# Assume light_view_map is the 2D array of leaf node indices.

def sparse_octree_filtering_with_node_map(node_map, octree, light_view_map, mip_level):
    # Get dimensions of the lowest level of the node map
    width, height = node_map[0].shape
    
    # Launch one thread per pixel of the lowest node map level
    threads = []
    for y in range(height):
        for x in range(width):
            threads.append(Thread(target=filter_thread, args=(x, y, node_map, octree, light_view_map, mip_level)))
            
    for thread in threads:
        thread.start()
        
    for thread in threads:
        thread.join()

def filter_thread(x, y, node_map, octree, light_view_map, mip_level):
    # 1. Get the leaf node index from the light-view map for this pixel
    leaf_node_idx = light_view_map[y, x]
    
    # 2. Find the target node in the octree for the specified MIP-map level
    # This involves traversing up from the leaf node to the desired level
    target_node = octree.get_node_at_level(leaf_node_idx, mip_level)
    
    # 3. Check the current level of the node map for common ancestors
    # The node map at the current mip_level stores LCAs for the previous level
    # We need to check if another thread has already claimed this target_node
    
    # Determine the position in the node map corresponding to the current mip_level
    # Assuming node_map is a list of 2D arrays, where index 0 is lowest level
    current_node_map_level = node_map[mip_level]
    
    # Check if this node has been marked as processed by another thread
    # The heuristic checks if the node map indicates a common ancestor
    # If multiple threads map to the same node, only the "upper left" one proceeds
    
    # To implement "upper left" tie-breaking, we check if there is another thread
    # with a smaller (x, y) coordinate that targets the same node.
    # In a parallel implementation, this requires synchronization or a shared state.
    # For simplicity, we assume a shared 'processed_nodes' set or atomic check.
    
    is_upper_left = True
    # Check if any other thread with x' < x or (x' == x and y' < y) targets the same node
    # This part is complex to implement purely in pseudocode without a shared data structure
    # [G] In practice, this is often implemented using atomic operations on a shared 
    # 'processed' flag or by checking a shared 'node_owner' map.
    
    # Simplified logic: Check if the node map at the current level indicates 
    # that this specific node is already being handled by a "higher priority" thread.
    # The node map itself helps identify *which* node is the LCA, but the 
    # "upper left" rule requires comparing thread coordinates.
    
    # Let's assume a shared dictionary 'node_to_thread' that maps node_idx to (x, y)
    # If node_idx is not in node_to_thread, or if current (x,y) is upper-left of stored (x',y'),
    # then this thread proceeds.
    
    if target_node not in node_to_thread:
        node_to_thread[target_node] = (x, y)
    else:
        owner_x, owner_y = node_to_thread[target_node]
        if (x, y) > (owner_x, owner_y):
            # This thread is not the upper-left one, terminate
            return
    
    # 4. Compute the filtered value for the target node
    filtered_value = compute_mip_value(octree, target_node, mip_level)
    
    # 5. Store the result
    octree.set_mip_value(target_node, mip_level, filtered_value)
```

**Parameters**
*   **MIP Level:** The current level of the MIP-map being generated. Typical values depend on the octree depth.
*   **Node Map Levels:** The number of levels in the node map pyramid corresponds to the number of MIP-map levels.

**Edge Cases and Pitfalls**
*   **Precision:** Not given in the sources.
*   **Boundaries:** The "upper left" tie-breaking rule must be consistently defined to avoid race conditions or missed computations [S1, S2].
*   **Degenerate Input:** If the light-view map is empty, no threads are launched. If the octree has no nodes at the target level, the lookup fails.

## Subtopics

### [Anisotropic Voxel Filtering for Mipmap Generation](2996-anisotropic-voxel-filtering-for-mipmap-generation.md)
This method filters lighting data at the leaf nodes of an octree anisotropically along the positive and negative XYZ directions to prepare for cone tracing. It stores six directional values per voxel to preserve directional information.

### [Distribution over Levels (MIP-mapping)](3045-distribution-over-levels-mip-mapping.md)
Distribution over Levels (MIP-mapping) filters radiance values from the lowest level of a sparse voxel octree and stores the results in higher levels to form a MIP-map hierarchy. It differs from regular grid MIP-mapping by explicitly handling the boundary sharing of 3x3x3 voxel bricks in an octree structure, requiring weighted averaging to account for overlapping voxels.

## Operating characteristics
*   **Time Complexity:** The method reduces the number of threads launched, leading to a speedup of > 2x compared to not using the heuristic [S1, S2]. The complexity per thread remains similar, but the total work is reduced.
*   **Memory Complexity:** Requires additional memory for the 2D node map, which is a MIP-map pyramid of the same dimensions as the light-view map [S1, S2].
*   **Measured Performance:** Yields > 2x speedup compared to not using the heuristic to terminate threads [S1, S2].
*   **Strengths:** Significantly reduces redundant computation in parallel MIP-map generation for sparse octrees [S1, S2].
*   **Limitations:** Requires the construction and maintenance of the 2D node map [S1, S2]. The "upper left" tie-breaking rule adds complexity to the synchronization logic [S1, S2].
*   **When to Use:** When generating MIP-maps for sparse voxel octrees in a parallel environment where redundant thread execution is a bottleneck [S1, S2].
*   **When Not to Use:** When memory is constrained, or when the overhead of maintaining the node map outweighs the benefits of thread termination (e.g., very small octrees).

## Background
The method was introduced in the paper "Interactive Indirect Illumination Using Voxel Cone Tracing" by Bensen et al. [S1, S2]. It is part of the voxel cone tracing technique for real-time global illumination. The node map heuristic was developed to optimize the MIP-map generation step, which is critical for the performance of voxel cone tracing [S1, S2].

## Sources
[S1] Interactive Indirect Illumination Using Voxel Cone Tracing, https://doi.org/10.1111/j.1467-8659.2011.02063.x, fetched 2026-10-26
[S2] PDF Interactive Indirect Illumination Using Voxel Cone Tracing, https://research.nvidia.com/sites/default/files/publications/GIVoxels-pg2011-authors.pdf, fetched 2026-10-27
