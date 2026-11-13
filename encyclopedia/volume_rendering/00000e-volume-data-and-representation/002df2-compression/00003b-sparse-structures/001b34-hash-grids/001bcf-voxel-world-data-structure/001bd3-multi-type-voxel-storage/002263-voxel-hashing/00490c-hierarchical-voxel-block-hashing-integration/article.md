# Hierarchical Voxel Block Hashing Integration

Also called: Parallel Hash Table Allocation, TSDF Integration with Hierarchical Blocks

## Overview
Hierarchical Voxel Block Hashing Integration is a method for efficiently integrating depth images into a 3D Truncated Signed Distance Field (TSDF) by using a multi-resolution spatial hash table. It solves the problem of memory redundancy and slow lookup in large-scale dense reconstruction by storing active voxel data in coarse blocks that can be refined to finer levels only where necessary. Unlike flat voxel hashing which allocates memory for every active voxel uniformly, this method maintains a hierarchy of hash tables, allowing $O(1)$ average-case access to voxel data while significantly reducing the memory footprint for sparse or partially observed scenes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A depth image $I_D$, camera pose $(\mathbf{R}, \mathbf{t})$, intrinsic matrix $\mathbf{K}$, and the current state of the hierarchical hash table structure.
*   **Output:** Updated TSDF values $F(\mathbf{X})$ and observation weights $W(\mathbf{X})$ for all voxels $\mathbf{X}$ intersected by the viewing frustum.

**Data Structures:**
The method uses a hierarchy of hash tables, one for each resolution level.
*   **Hash Table Layout:** Each level $l$ has a hash table mapping block coordinates to a linked list of voxel blocks.
*   **Voxel Block:** A contiguous chunk of voxels. The block contains the TSDF values $F$ and weights $W$ for its constituent voxels.
*   **Linked List Entry:** Stores a pointer to the next block in the hash bucket and the block's coordinates.
*   **Pools:** Two pools are maintained for parallel allocation:
    1.  Pool of empty voxel blocks.
    2.  Pool of linked list entries.

**Parameters:**
*   Base voxel size $s = 2$ mm [S1].
*   Number of resolution levels: 3 [S1].
*   Coarsest level voxel size: 8 mm [S1].
*   TSDF truncation band: 24 mm [S1].

**Algorithm Steps:**

1.  **Ray Casting and Block Lookup:**
    For each pixel in the depth image, compute the corresponding 3D line segment $L$. Determine the set of blocks $B$ at the coarsest level of the hierarchy that intersect this segment.
    For each block $b \in B$:
    a. Compute the hash value for $b$.
    b. Check if the hash table at the current level contains an entry for $b$.
    c. If the block exists, proceed to integration.
    d. If the block does not exist, check if the hash table indicates that information exists at a finer level. If so, proceed to the next hierarchy level.
    e. If no information exists at any finer level, allocate a new block from the pool and insert it into the hash table.

2.  **Parallel Allocation (GPU):**
    To enable parallel execution without critical sections, split allocation into two stages:
    a. **Stage 1 (Marking):** Each thread identifies blocks that need allocation. Mark the corresponding buckets in the hash tables and store the coordinates of these blocks.
    b. **Stage 2 (Modification):** Threads modify the hash tables. For marked buckets, start new linked lists or extend existing ones using atomic operations. Use the pre-maintained pools for voxel blocks and linked list entries.
    c. Collect the list of newly allocated or existing blocks that contain novel depth information.

3.  **TSDF Integration:**
    Iterate through the list of observed voxel blocks. For each voxel $\mathbf{X}$ in a block:
    a. Project the voxel center into the depth image to retrieve the observed depth $d$.
    b. The projection is calculated as:
       $$ d = I_D(\pi(\mathbf{K}(\mathbf{R}\mathbf{X} + \mathbf{t}))) $$
       where $\pi$ normalizes homogeneous 2D coordinates to inhomogeneous ones, $\mathbf{R}$ and $\mathbf{t}$ are the rotation and translation of the camera, and $\mathbf{K}$ is the intrinsic matrix [S1].
    c. If the voxel projects into the depth image bounds and has a valid depth measurement:
       i. Update the weighted sum for the TSDF value $F(\mathbf{X})$:
          $$ F(\mathbf{X}) := \frac{W(\mathbf{X})F(\mathbf{X}) + d}{W(\mathbf{X}) + 1} $$
       ii. Increment the observation weight $W(\mathbf{X})$:
          $$ W(\mathbf{X}) := W(\mathbf{X}) + 1 $$
       Note: The standard TSDF truncation is applied to $F(\mathbf{X})$ after update, clamping values outside the truncation band [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def integrate_depth_image(depth_image, camera_pose, intrinsics, hash_tables, block_pool, list_pool):
    """
    Integrates a single depth frame into the hierarchical voxel structure.
    
    Args:
        depth_image: 2D array of depth values.
        camera_pose: Tuple (R, t) representing camera rotation and translation.
        intrinsics: Camera intrinsic matrix K.
        hash_tables: List of hash tables, one per resolution level.
        block_pool: Pool of available voxel blocks.
        list_pool: Pool of available linked list entries.
        
    Returns:
        Updated hash_tables and pools.
    """
    R, t = camera_pose
    K = intrinsics
    
    # Stage 1: Mark buckets needing allocation
    blocks_to_allocate = []
    for y in range(depth_image.height):
        for x in range(depth_image.width):
            # Compute ray direction from pixel (x, y)
            ray_dir = unproject_pixel(x, y, K)
            # Determine blocks intersected by ray at coarsest level
            blocks = get_intersected_blocks(ray_dir, camera_origin, hash_tables[0])
            
            for block in blocks:
                hash_val = compute_hash(block.coords, hash_tables[0])
                if not hash_tables[0].contains(hash_val):
                    # Check finer levels
                    if not check_finer_levels(block.coords, hash_tables):
                        blocks_to_allocate.append((block, 0)) # Level 0
                    else:
                        # Proceed to finer level logic (simplified for pseudocode)
                        pass
                else:
                    # Block exists, mark for integration later
                    pass

    # Stage 2: Modify hash tables using atomic operations
    for block, level in blocks_to_allocate:
        hash_val = compute_hash(block.coords, hash_tables[level])
        # Atomic check-and-swap or similar to allocate from pool
        new_entry = list_pool.pop()
        new_block = block_pool.pop()
        new_entry.block = new_block
        new_entry.next = hash_tables[level].get(hash_val)
        hash_tables[level].set(hash_val, new_entry)

    # Integration Phase
    for level in range(len(hash_tables)):
        for bucket in hash_tables[level].buckets:
            entry = bucket.head
            while entry:
                block = entry.block
                for voxel in block.voxels:
                    # Project voxel center to depth image
                    world_pos = voxel.center + t
                    cam_pos = R @ world_pos
                    hom_2d = K @ cam_pos
                    pix_2d = hom_2d[:2] / hom_2d[2]
                    
                    if is_valid_pixel(pix_2d, depth_image):
                        depth_val = depth_image[pix_2d.y, pix_2d.x]
                        # Update TSDF
                        old_F = voxel.F
                        old_W = voxel.W
                        voxel.W = old_W + 1
                        voxel.F = (old_W * old_F + depth_val) / voxel.W
                        # Apply truncation
                        voxel.F = clamp(voxel.F, -truncation_band, truncation_band)
                
                entry = entry.next
```

## Subtopics

### [Volumetric Triangle Representation with Spatial Hashing](13835-volumetric-triangle-representation-with-spatial-hashing.md)
This method provides a compact, incremental data structure for real-time dense 3D reconstruction by embedding mesh vertices directly into a volumetric grid via spatial hashing. It solves the problem of memory redundancy and inefficient vertex querying in loose mesh storage strategies by establishing a one-to-one correspondence between voxel edges and mesh vertices. Unlike hierarchical approaches that store TSDF at the voxel level and triangles at the block level, this method maintains both geometric primitives (triangles) and scalar fields (TSDF) at the fine cube scale, enabling $O(1)$ vertex access and eliminating duplicate vertex allocation.

## Operating characteristics
*   **Time Complexity:** Lookup and integration are $O(1)$ on average per voxel due to hash table access. The hierarchical structure reduces the number of blocks to check at finer levels.
*   **Memory Complexity:** Significantly lower than flat voxel hashing for sparse scenes, as memory is only allocated for active blocks and refined only where needed. The use of pools and linked lists adds some overhead but allows efficient parallel allocation.
*   **Performance:** The two-stage allocation process enables efficient GPU parallelization with minimal synchronization overhead [S1].
*   **Strengths:** Efficient memory usage for large, sparse environments; fast parallel integration; supports multi-resolution details.
*   **Limitations:** Hash collisions can degrade performance (though mitigated by linked lists); requires careful tuning of truncation and voxel sizes for specific applications [S1].

## Background
The method was introduced in "Hierarchical Voxel Block Hashing for Efficient Integration of Depth Images" [S1]. It builds upon earlier work in voxel block hashing [12] and TSDF integration [1]. The approach was designed to address the scalability issues of dense 3D reconstruction in large-scale environments, particularly for real-time applications on GPU hardware. It is used in systems like the one described in the source, which utilizes online ICP tracking for camera pose estimation [S1].

## Sources
[S1] Hierarchical Voxel Block Hashing for Efficient Integration of Depth Images, https://doi.org/10.1109/lra.2015.2512958, fetched 2026-11-08
