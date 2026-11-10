# Hierarchical Voxel Block Hashing Integration

Also called: Parallel Hash Table Allocation, TSDF Integration with Hierarchical Blocks

## Overview
Hierarchical Voxel Block Hashing Integration is a method for efficiently integrating depth images into a 3D Truncated Signed Distance Field (TSDF) by using a multi-resolution spatial hash table. It solves the problem of memory inefficiency and cache misses in single-resolution voxel hashing by allocating fine-grained voxel data only where necessary, while maintaining a coarse grid for fast traversal. It sets itself apart from standard Voxel Hashing by introducing a hierarchy of resolution levels, allowing the system to skip empty regions at coarse levels and only allocate memory for active blocks at finer levels, thereby reducing memory footprint and improving parallel allocation throughput on GPUs.

## How it works
**Inputs and Outputs:**
- **Inputs:** A depth image $I_D$, camera intrinsics $\mathbf{K}$, camera pose $(\mathbf{R}, \mathbf{t})$, and the current hierarchical hash table state.
- **Outputs:** Updated TSDF values $F(\mathbf{X})$, observation counts $W(\mathbf{X})$, and modified hash tables with newly allocated blocks.

**Data Structures:**
The method maintains a hierarchy of hash tables, one for each resolution level.
- **Hash Table Entry:** Each bucket in the hash table stores a pointer to a linked list of voxel blocks. The linked list nodes contain coordinates of the block and a pointer to the next node.
- **Voxel Block Pool:** A pool of pre-allocated empty voxel blocks is maintained to enable fast, lock-free allocation.
- **Linked List Entry Pool:** A pool of linked list nodes is maintained to enable fast, lock-free insertion into hash table buckets.
- **Voxel Data:** Each voxel stores a TSDF value $F(\mathbf{X})$ and an observation weight $W(\mathbf{X})$.

**Parameters:**
- Base voxel size $s = 2$ mm [S1].
- Truncation band for TSDF = 24 mm [S1].
- Number of resolution levels = 3 [S1].
- Coarsest level voxel size = 8 mm [S1].

**Algorithm Steps:**

1. **Ray Casting and Block Lookup:**
   For each pixel in the depth image, compute the 3D line segment $L$ passing through the camera center and the pixel. Determine the set of voxel blocks $B$ at the coarsest level that intersect this line segment.
   For each block $b \in B$:
   - Compute the hash value for $b$.
   - Check if the block is already associated with voxel data in the hash table.
   - If the block exists, proceed to the next block.
   - If the block does not exist, mark it for allocation.
   - If the hash table at the current level indicates that information exists at a finer level, proceed to the next hierarchy level and repeat the lookup for the finer blocks.

2. **Parallel Allocation (Two-Stage):**
   To enable efficient GPU parallelization, the allocation of new blocks is split into two stages using atomic operations:
   - **Stage 1 (Marking):** Mark the buckets in the hash tables that require new blocks. Store the coordinates of the corresponding blocks in a temporary list. This stage identifies which blocks need to be allocated without modifying the hash structure yet.
   - **Stage 2 (Modification):** Modify the hash tables by starting new linked lists at the marked buckets or extending existing lists. Use the pools of empty voxel blocks and linked list entries to perform these allocations. This stage ensures that all parallel threads can allocate blocks without critical sections.

3. **Depth Integration:**
   After all required voxel blocks are allocated, iterate through the list of observed voxel blocks that contain novel depth information. For each voxel $\mathbf{X}$ in these blocks:
   - Project the voxel center into the depth image to retrieve the observed depth $d$. The projection is calculated as:
     $$ d = I_D(\pi(\mathbf{K}(\mathbf{R}\mathbf{X} + \mathbf{t}))) $$
     where $\pi$ normalizes homogeneous 2D coordinates to inhomogeneous ones, $\mathbf{K}$ is the camera intrinsic matrix, $\mathbf{R}$ is the rotation matrix, and $\mathbf{t}$ is the translation vector.
   - If the voxel projects into the depth image and has a valid depth measurement, update the TSDF value $F(\mathbf{X})$ and the observation weight $W(\mathbf{X})$ using the following weighted average formulas:
     $$ F(\mathbf{X}) := \frac{W(\mathbf{X})F(\mathbf{X}) + d}{W(\mathbf{X}) + 1} $$
     $$ W(\mathbf{X}) := W(\mathbf{X}) + 1 $$
   - Note: The standard TSDF truncation is applied to $F(\mathbf{X})$ after update, though the specific truncation function is not detailed in the provided notes beyond the truncation band value [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
BASE_VOXEL_SIZE = 2.0  # mm
TRUNCATION_BAND = 24.0 # mm
NUM_LEVELS = 3
COARSEST_VOXEL_SIZE = 8.0 # mm

# Data Structures
class VoxelBlock:
    def __init__(self):
        self.data = {} # Map from local voxel coordinates to (F, W)

class HashTable:
    def __init__(self, size):
        self.buckets = [None] * size # Linked list heads
        self.size = size

    def hash(self, block_coords):
        # Simple hash function for block coordinates
        return hash(block_coords) % self.size

    def get_or_create_block(self, block_coords, block_pool, list_pool):
        h = self.hash(block_coords)
        # Stage 1: Check if block exists
        current = self.buckets[h]
        while current is not None:
            if current.block_coords == block_coords:
                return current.voxel_block, False # Exists
            current = current.next
        # Stage 2: Allocate new block
        if block_pool.empty():
            raise Exception("Block pool empty")
        new_block = block_pool.pop()
        new_list_node = list_pool.pop()
        new_list_node.block_coords = block_coords
        new_list_node.voxel_block = new_block
        new_list_node.next = self.buckets[h]
        self.buckets[h] = new_list_node
        return new_block, True # Newly allocated

def integrate_depth_image(depth_image, K, R, t, hash_tables, block_pools, list_pools):
    # Step 1: Identify blocks to allocate
    blocks_to_allocate = []
    
    # Iterate over pixels to find intersecting blocks at coarsest level
    for pixel in depth_image:
        # Compute ray direction from pixel
        ray_dir = unproject_pixel(pixel, K)
        ray_origin = camera_center(R, t)
        
        # Determine blocks at coarsest level intersected by ray
        coarsest_blocks = get_intersecting_blocks(ray_origin, ray_dir, COARSEST_VOXEL_SIZE)
        
        for block_coords in coarsest_blocks:
            # Check hash table at coarsest level
            ht = hash_tables[0]
            block, allocated = ht.get_or_create_block(block_coords, block_pools[0], list_pools[0])
            if allocated:
                blocks_to_allocate.append((block_coords, 0)) # Level 0
            
            # If allocated, check if finer levels are needed
            # If hash table indicates info at finer level, proceed to next level
            # For simplicity, assume we always check finer levels if block is new or if we want to integrate
            # The source says: "If the hash table at the coarsest level indicates that information 
            # about the voxel is present at a finer level, we proceed in the same fashion on the next hierarchy level."
            # This implies a recursive or iterative check.
            
            # For this pseudocode, we assume we iterate through all levels for new blocks
            # or if the block exists, we might still need to integrate if it's a new depth observation.
            # The source implies allocation happens if block is not present.
            
            # Let's refine: Only allocate if not present. If present, we might still integrate.
            # But the integration step iterates over "observed voxel blocks that contain novel depth information".
            # So we need to track which blocks have novel data.
            
            # Simplified: If allocated, add to list. If not allocated, check if we need to go deeper.
            # The source suggests checking finer levels only if the coarse block indicates finer info exists.
            # This is a lookup optimization.
            
            # For implementation, we will assume we check all levels for the ray's path.
            for level in range(1, NUM_LEVELS):
                # Compute finer block coordinates
                finer_block_coords = get_finer_block_coords(block_coords, level)
                ht = hash_tables[level]
                finer_block, finer_allocated = ht.get_or_create_block(finer_block_coords, block_pools[level], list_pools[level])
                if finer_allocated:
                    blocks_to_allocate.append((finer_block_coords, level))

    # Step 2: Integrate depth into allocated blocks
    for block_coords, level in blocks_to_allocate:
        # Get voxel size for this level
        voxel_size = COARSEST_VOXEL_SIZE / (2 ** level)
        
        # Get the block from the hash table
        ht = hash_tables[level]
        h = ht.hash(block_coords)
        current = ht.buckets[h]
        while current is not None:
            if current.block_coords == block_coords:
                voxel_block = current.voxel_block
                break
            current = current.next
        
        # Iterate over voxels in the block
        # The source says: "iterate through the list of observed voxel blocks containing novel depth information"
        # This implies we only integrate voxels that were hit by the ray and have valid depth.
        
        # For each voxel in the block that was intersected by the ray
        # We need to know which voxels were intersected. The ray casting step should return this.
        # Let's assume we have a list of voxels intersected by the ray for this block.
        intersected_voxels = get_intersected_voxels_in_block(ray_origin, ray_dir, block_coords, voxel_size)
        
        for voxel_local_coords in intersected_voxels:
            voxel_global_coords = block_coords * (voxel_size * 2) + voxel_local_coords * voxel_size
            
            # Project voxel to depth image
            voxel_homogeneous = [voxel_global_coords[0], voxel_global_coords[1], voxel_global_coords[2], 1.0]
            camera_coords = R @ voxel_homogeneous[:3] + t
            pixel_homogeneous = K @ camera_coords
            pixel_2d = pixel_homogeneous[:2] / pixel_homogeneous[2]
            
            # Check if pixel is within image bounds and has valid depth
            if is_valid_pixel(pixel_2d, depth_image.shape):
                depth_value = depth_image[int(pixel_2d[1]), int(pixel_2d[0])]
                if depth_value > 0: # Valid depth
                    # Update TSDF
                    # F(X) and W(X) are stored in voxel_block.data[voxel_local_coords]
                    if voxel_local_coords not in voxel_block.data:
                        voxel_block.data[voxel_local_coords] = (0.0, 0.0)
                    
                    F, W = voxel_block.data[voxel_local_coords]
                    # Weighted average
                    new_F = (W * F + depth_value) / (W + 1)
                    new_W = W + 1
                    
                    # Apply truncation
                    # Truncation band is 24mm. TSDF is typically normalized by truncation band.
                    # The source formula uses raw depth. The truncation is likely applied to the distance.
                    # Standard TSDF: F = min(max(distance, -truncation), truncation) / truncation
                    # The source formula (3) is a weighted average of depth. This is a specific variant.
                    # We will follow the source formula exactly.
                    
                    voxel_block.data[voxel_local_coords] = (new_F, new_W)
                    
                    # Apply truncation to F
                    # Assuming F is the signed distance. The source doesn't specify the sign convention 
                    # or the truncation application in the formula, only the band value.
                    # We assume standard TSDF truncation is applied after the weighted average.
                    # F = max(-TRUNCATION_BAND, min(TRUNCATION_BAND, F)) / TRUNCATION_BAND
                    # However, the source formula (3) uses depth directly. 
                    # It is likely that F represents the distance to the surface, and truncation is applied.
                    # Given the ambiguity, we will store the raw weighted average and apply truncation.
                    # But the source says "update a weighted sum" and gives the formula.
                    # We will assume F is the TSDF value and truncation is applied externally or implicitly.
                    # For implementation, we will apply truncation to F.
                    
                    F_truncated = max(-TRUNCATION_BAND, min(TRUNCATION_BAND, new_F)) / TRUNCATION_BAND
                    voxel_block.data[voxel_local_coords] = (F_truncated, new_W)
```

## Operating characteristics
- **Time Complexity:** The lookup and allocation are $O(1)$ on average per block due to hashing. The integration is proportional to the number of voxels intersected by the rays. The two-stage allocation allows for efficient parallelization on GPUs with minimal synchronization overhead.
- **Memory Complexity:** Memory usage is proportional to the number of active voxel blocks, which is significantly less than a dense voxel grid. The hierarchical structure further reduces memory usage by avoiding allocation of fine-grained voxels in empty regions.
- **Measured Performance:** The method was tested on the "7 Scenes" dataset and custom sequences. It provides high detail with a base voxel size of 2 mm and a truncation band of 24 mm. The hierarchical representation with 3 levels (coarsest 8 mm) was shown to be more efficient than standard voxel block hashing [S1].
- **Strengths:** Efficient memory usage, fast parallel allocation, high detail resolution.
- **Limitations:** Requires careful tuning of parameters (voxel size, truncation band, number of levels). The hash table may suffer from collisions if not sized appropriately.

## Background
The method was introduced in the paper "Hierarchical Voxel Block Hashing for Efficient Integration of Depth Images" [S1]. It builds on the concept of voxel block hashing [12] and TSDF integration [1]. The hierarchical approach was proposed to address the limitations of single-resolution voxel hashing, particularly in terms of memory efficiency and performance for large-scale scenes. The method is implemented using CUDA for GPU acceleration [S1].

## Sources
[S1] Hierarchical Voxel Block Hashing for Efficient Integration of Depth Images, https://doi.org/10.1109/lra.2015.2512958, fetched 2026-11-08
