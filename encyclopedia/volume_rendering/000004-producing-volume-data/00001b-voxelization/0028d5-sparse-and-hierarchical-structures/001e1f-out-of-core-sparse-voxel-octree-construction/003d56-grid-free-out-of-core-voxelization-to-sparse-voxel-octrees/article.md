# Grid-Free Out-Of-Core Voxelization to Sparse Voxel Octrees

Also called: GPU-based Out-of-Core SVO Builder without Atomics, Stream-based Out-of-Core SVO Construction

## Overview
A stream-based out-of-core voxelization technique that processes triangles and voxels in batches to build Sparse Voxel Octrees (SVOs) on the GPU, avoiding grid-based structures and atomic operations by predicting voxel counts and using Morton ordering [S1]. It differs from grid-based out-of-core methods by maintaining no global occupancy grid, instead using a Morton queue to manage voxel locality across batch boundaries [S1].

## How it works
**Inputs:** Triangle mesh (vertices, indices), user-defined max voxel count per batch ($V_{max}$), user-defined subdivision threshold ($T_{sub}$).
**Outputs:** Sparse Voxel Octree (SVO) with attributes.

**Data Structures:**
1.  **Triangle Batch:** List of triangles.
2.  **Per-Triangle Voxel Memory:** Array of voxel-attribute pairs. Size determined by prediction.
3.  **Morton Queue:** Stores voxel-attribute pairs from current and previous iterations. Sorted by Morton index.
4.  **SVO Node Buffer:** Bitmasks for occupancy, arrays for attributes and child pointers.

**Steps:**

1.  **Triangle Homogenization:** Subdivide triangles to balance GPU workload (1 triangle per thread) and ensure Morton locality [S1].
    *   Calculate ratio: $R = \frac{\text{Area of largest face of bounding box}}{\text{Area of a voxel face}}$.
    *   If $R > T_{sub}$, subdivide based on triangle shape [S1]:
        *   Case 1: Long thin triangles (angles $>90^\circ$ or $<20^\circ$).
        *   Case 2: Other triangles.
        *   Case 3: Specific geometric handling for degenerate cases.
    *   [G] Ensure subdivision maintains manifold connectivity if required by downstream rendering.

2.  **Sorting:** Sort triangles by the minimum Morton index of their bounding boxes. This determines the earliest possible need for a triangle in SVO creation [S1].

3.  **Batch Creation:** Create batches such that the predicted voxel count per batch does not exceed $V_{max}$. Store the minimum Morton index of the first triangle of the *next* batch ($M_{next}$) to define the valid Morton range for SVO node creation in the current batch [S1].

4.  **Voxelization:** Voxelize each triangle batch into per-triangle voxel memory using offsets given by the prediction. This avoids atomic operations [S1].
    *   Use conservative surface voxelization method of Schwarz and Seidel [S1].
    *   Each GPU thread processes one triangle.
    *   Create attributes by projecting voxel centers to UV coordinates [S1].
    *   Remove placeholders and copy valid voxel-attribute pairs to the Morton Queue [S1].

5.  **Morton Queueing:** The Morton Queue stores unusable voxels from previous iterations and all voxels from the current iteration. Sort the Morton Queue [S1]. Extract voxel-attribute pairs for SVO creation using $M_{next}$ as the stop index [S1].

6.  **SVO Creation:** Create SVO parts bottom-up using parallel stream compaction. Each GPU thread processes all child nodes of one parent (parent index = child index / 8) [S1].
    *   Use a data structure similar to Laine and Karras [S1]:
        *   Bitmasks to address non-empty voxels (mod).
        *   Voxel attributes (e.g., color).
        *   Indices (child-pointer) on CPU [S1].

7.  **Post-Order Attribute Creation:** Create parent attributes only if all child nodes are available. Use a stitch queue on each hierarchy level to buffer voxel-attribute pairs until all nodes are given. Attributes can be determined by multipass operations [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def grid_free_out_of_core_voxelization(triangles, V_max, T_sub):
    # Step 1: Homogenization
    homogenized_triangles = []
    for tri in triangles:
        bbox = compute_bounding_box(tri)
        largest_bbox_face_area = max_area_face(bbox)
        voxel_face_area = voxel_size ** 2
        ratio = largest_bbox_face_area / voxel_face_area
        
        if ratio > T_sub:
            subdivided = subdivide_triangle(tri, ratio, T_sub)
            homogenized_triangles.extend(subdivided)
        else:
            homogenized_triangles.append(tri)
            
    # Step 2: Sorting
    sorted_triangles = sort_by_min_morton_index(homogenized_triangles)
    
    # Step 3: Batch Creation
    batches = create_batches(sorted_triangles, V_max)
    
    # Initialize Morton Queue
    morton_queue = []
    
    for i, batch in enumerate(batches):
        # Step 4: Voxelization
        per_triangle_voxels = []
        for tri in batch:
            # Predict voxel count to allocate memory without atomics
            predicted_count = predict_voxel_count(tri)
            # Voxelize using Schwarz and Seidel method
            voxels = conservative_voxelize(tri, predicted_count)
            # Project voxel centers to UV for attributes
            for v in voxels:
                v.attributes = project_to_uv(v.center, tri.uv_coords)
            per_triangle_voxels.extend(voxels)
            
        # Remove placeholders and add to Morton Queue
        valid_voxels = remove_placeholders(per_triangle_voxels)
        morton_queue.extend(valid_voxels)
        
        # Step 5: Morton Queueing
        # Determine stop index from next batch
        if i + 1 < len(batches):
            next_batch_first_tri = batches[i+1][0]
            stop_morton = compute_min_morton_index(next_batch_first_tri.bbox)
        else:
            stop_morton = float('inf')
            
        # Sort Morton Queue
        morton_queue.sort(key=lambda v: v.morton_index)
        
        # Extract valid voxel-attribute pairs for SVO creation
        current_batch_voxels = [v for v in morton_queue if v.morton_index < stop_morton]
        
        # Remove processed voxels from queue
        morton_queue = [v for v in morton_queue if v.morton_index >= stop_morton]
        
        # Step 6: SVO Creation
        # Create SVO parts bottom-up
        svos = create_svo_bottom_up(current_batch_voxels)
        
        # Step 7: Post-Order Attribute Creation
        # Use stitch queue to buffer until all children available
        finalize_svo_attributes(svos)
        
    return final_svo
```

## Operating characteristics
*   **Time Complexity:** Dominated by sorting triangles and the Morton Queue. Parallel voxelization and SVO creation are efficient on GPU.
*   **Memory Complexity:** Out-of-core. Memory usage is logarithmic in the size of the SVO, limited by $V_{max}$ and the Morton Queue size [S1].
*   **Performance:** For the Hairball scene (2.8M triangles) at 2048 resolution, the algorithm takes 274.4s, compared to 4096s for Laine/Karras 2011 and 81.0s for Baert et al. 2014 [S1]. For Lucy (28.0M triangles) at 2048 resolution, it takes 134.4s, compared to 964.3s for Laine/Karras 2011 and 17.5s for Baert et al. 2014 [S1].
*   **Strengths:** No atomic operations required. Efficient GPU parallelism. Out-of-core capability.
*   **Limitations:** Requires careful prediction of voxel counts. Sorting overhead.

## Background
The method was presented in August 2015 [S1]. It builds on [Out-of-Core Sparse Voxel Octree Construction](1689-out-of-core-sparse-voxel-octree-construction.md). It references [Crassin & Green 2012], [Laine & Karras 2010+2011], [Baert et al. 2013+2014], and [Schwarz and Seidel 2010] [S1].

## Sources
[S1] PDF Grid-Free Out-Of-Core Voxelization to Sparse Voxel Octrees on GPU, https://www.highperformancegraphics.org/wp-content/uploads/2015/Papers-Session4/grid_free_out_of_core_voxelization.pdf, fetched 2026-10-29
