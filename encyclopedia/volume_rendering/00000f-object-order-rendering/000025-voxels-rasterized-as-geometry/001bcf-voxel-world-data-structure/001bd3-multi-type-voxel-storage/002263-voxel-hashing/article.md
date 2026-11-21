# Voxel Hashing

Also called: Hash-based voxel representation, Sparse TSDF Representation

## Overview
Voxel Hashing is a sparse volumetric data structure that enables real-time 3D scene reconstruction by allocating memory only where measured surfaces exist, rather than filling an entire fixed grid [S3, unofficial]. It pairs small blocks of voxels with a hash table, giving systems constant-time access to surface data while keeping memory usage proportional to the actual geometry observed [S3, unofficial]. It sets itself apart from its nearest alternative, [KinectFusion](52-kinectfusion.md), which uses a rigid, pre-allocated 3D grid confined to a small volume, by allowing room-scale and larger scenes with dynamic allocation [S3, unofficial]. Compared to [Octree Methods](1399-octree-methods.md), it offers $O(1)$ hash lookup and better GPU parallelism by avoiding tree traversal overhead and pointer chasing [S1].

## How it works
**Inputs:** A stream of depth frames from a sensor (structured light, time of flight, or stereo vision) and the camera pose for each frame [S3, unofficial].
**Outputs:** A sparse Truncated Signed Distance Field (TSDF) volume stored in a hash table, and optionally a triangle mesh extracted via Marching Cubes [S3, unofficial].

**Data Structures:**
1. **Voxel Block:** A fixed-size 3D grid of voxels [S3, unofficial]. The standard block size is $8 \times 8 \times 8$ voxels [S2][S3, unofficial].
2. **Voxel:** Contains a truncated signed distance value ($d$) and a confidence weight ($w$) [S3, unofficial].
3. **Hash Table:** A flat lookup table storing pointers or indices to allocated Voxel Blocks [S3, unofficial]. It uses open addressing with linear probing to handle collisions [S3, unofficial]. The table capacity is typically sized to keep the load factor under 50% [S3, unofficial].

**Parameters:**
* **Voxel Block Size:** $8 \times 8 \times 8$ [S2][S3, unofficial].
* **TSDF Truncation Band:** Set to one block width (8 voxels) in original experiments [S2].
* **Voxel Resolution:** 5 mm for human sequences, 1 cm for larger scenes [S2].
* **Hash Table Capacity:** ~500,000 entries for room-scale scenes on 2013-era hardware [S3, unofficial].

**Algorithm Steps:**
1. **Capture Depth Data:** Acquire a depth frame $D_t$ and camera pose $P_t$ [S3, unofficial].
2. **Frustum Culling:** Determine which voxel blocks intersect the camera frustum and are close to measured surfaces [S3, unofficial].
3. **Hash Lookup and Allocation:** For each candidate block with integer coordinate $B_{xyz}$, compute hash index $h = \text{Hash}(B_{xyz})$. Use linear probing to find the first empty slot or the slot containing $B_{xyz}$. If the slot is empty, allocate a new Voxel Block and store it at $h$. If the slot contains a different block coordinate (collision), continue probing [S3, unofficial].
4. **TSDF Fusion:** For each voxel $v$ in the allocated block, compute the signed distance $d_{new}$ from the depth measurement. If $|d_{new}| < \text{TruncationBand}$, update the stored value using a running weighted average. Otherwise, the voxel remains unchanged [S3, unofficial].
5. **Garbage Collection (Optional):** Identify stale blocks that are no longer in the frustum or relevant to the current view and free them for reallocation [S3, unofficial].
6. **Mesh Extraction (Optional):** Run a Marching Cubes variant over all allocated blocks to generate a triangle mesh [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Constants
BLOCK_SIZE = 8
TRUNCATION_BAND = BLOCK_SIZE  # 8 voxels
VOXEL_RESOLUTION = 0.005      # 5 mm
HASH_TABLE_SIZE = 500000      # Power of 2 recommended for masking
LOAD_FACTOR_LIMIT = 0.5

# Data Structures
class VoxelBlock:
    def __init__(self):
        # 8x8x8 voxels, each storing distance and weight
        self.voxels = [0.0] * (BLOCK_SIZE ** 3)
        self.weights = [1.0] * (BLOCK_SIZE ** 3)

class HashTable:
    def __init__(self, capacity):
        self.capacity = capacity
        self.mask = capacity - 1
        # Stores block coordinates (x,y,z) or None if empty
        self.keys = [None] * capacity
        # Stores pointers to VoxelBlock objects or None
        self.values = [None] * capacity
        self.used_slots = 0

    def insert(self, key, value):
        idx = spatial_hash(*key) & self.mask
        while self.keys[idx] is not None:
            if self.keys[idx] == key:
                return False  # Already exists
            idx = (idx + 1) & self.mask
        self.keys[idx] = key
        self.values[idx] = value
        self.used_slots += 1
        return True

    def get_or_allocate(self, key, allocator):
        idx = spatial_hash(*key) & self.mask
        while self.keys[idx] is not None:
            if self.keys[idx] == key:
                return self.values[idx]
            idx = (idx + 1) & self.mask
        # Slot empty, allocate
        block = allocator()
        self.keys[idx] = key
        self.values[idx] = block
        self.used_slots += 1
        return block

    def is_full(self):
        return (self.used_slots / self.capacity) >= LOAD_FACTOR_LIMIT

def spatial_hash(x, y, z):
    # Deterministic spatial hash for 3D integer coordinates
    p1 = 73856093
    p2 = 19349663
    p3 = 83492791
    return (x * p1) ^ (y * p2) ^ (z * p3)

def world_to_voxel_coord(point, camera_pose, resolution):
    # Transform point to world space, then quantize to voxel grid
    world_point = camera_pose.transform(point)
    return (world_point.x / resolution, world_point.y / resolution, world_point.z / resolution)

def is_inside_block(voxel_coord, block_coord, block_size):
    local_x = voxel_coord.x - block_coord.x * block_size
    local_y = voxel_coord.y - block_coord.y * block_size
    local_z = voxel_coord.z - block_coord.z * block_size
    return 0 <= local_x < block_size and 0 <= local_y < block_size and 0 <= local_z < block_size

def voxel_to_local_index(voxel_coord, block_coord, block_size):
    local_x = int(voxel_coord.x - block_coord.x * block_size)
    local_y = int(voxel_coord.y - block_coord.y * block_size)
    local_z = int(voxel_coord.z - block_coord.z * block_size)
    return local_x + local_y * block_size + local_z * (block_size ** 2)

def fuse_voxel(new_dist, old_dist, old_weight):
    # Truncation check: only fuse if within band
    if abs(new_dist) > TRUNCATION_BAND:
        return old_dist, old_weight
    # Weighted average fusion
    new_weight = old_weight + 1.0
    new_dist_stored = (old_weight * old_dist + new_dist) / new_weight
    return new_dist_stored, new_weight

def process_frame(depth_frame, camera_pose, hash_table, block_allocator):
    # 1. Frustum Culling and Block Identification
    # Standard frustum-culling step to find candidate 8x8x8 blocks
    candidate_blocks = get_blocks_in_frustum(depth_frame, camera_pose, BLOCK_SIZE)
    
    for block_coord in candidate_blocks:
        # 2. Hash Lookup
        block = hash_table.get_or_allocate(block_coord, block_allocator)
        
        # 3. TSDF Fusion
        for point in depth_frame.points:
            voxel_coord = world_to_voxel_coord(point, camera_pose, VOXEL_RESOLUTION)
            if is_inside_block(voxel_coord, block_coord, BLOCK_SIZE):
                local_idx = voxel_to_local_index(voxel_coord, block_coord, BLOCK_SIZE)
                d_new = point.distance
                d_old = block.voxels[local_idx]
                w_old = block.weights[local_idx]
                
                d_new_stored, w_new_stored = fuse_voxel(d_new, d_old, w_old)
                block.voxels[local_idx] = d_new_stored
                block.weights[local_idx] = w_new_stored

def extract_mesh(hash_table):
    mesh = []
    for i in range(hash_table.capacity):
        if hash_table.values[i] is not None:
            block = hash_table.values[i]
            # Standard Marching Cubes variant implementation [G]
            triangles = marching_cubes_block(block)
            mesh.extend(triangles)
    return mesh

def get_blocks_in_frustum(depth_frame, camera_pose, block_size):
    # Sources do not specify the exact frustum culling implementation.
    # Standard approach: compute world-space bounding box of depth points,
    # then iterate over block coordinates intersecting the box.
    pass

def marching_cubes_block(block):
    # Standard Marching Cubes algorithm applied to the 8x8x8 voxel grid.
    # Returns a list of triangles. Implementation details omitted as they are standard.
    pass
```

**Pitfalls and Edge Cases:**
* **Hash Collisions:** Multiple block coordinates can resolve to the same hash bucket. Open addressing with linear probing handles this well on GPUs because it avoids pointer dereferencing. Keeping the table's load factor under 50% minimizes probe chain length and prevents thread stalls during parallel access [S3, unofficial].
* **Stale Blocks:** As the camera moves, previously scanned regions may fall out of relevance. Stale blocks accumulate and crowd out space for new data. A garbage collection pass identifies blocks whose surface information is no longer needed and frees them for reallocation [S3, unofficial].
* **Tracking Drift:** When the camera pose estimator loses accuracy, incoming depth data fuses into incorrect positions, producing ghosting or doubled surfaces. Mitigated by pairing with robust pose alignment methods such as frame-to-model Iterative Closest Point (ICP) or learned visual feature matching [S3, unofficial].
* **Moving Objects:** The standard pipeline assumes a static world. People walking through the scene or objects being relocated cause smeared geometry. Addressed by extensions like DynamicFusion and VolumeDeform, which warp the volume to accommodate non-rigid motion [S3, unofficial].
* **Fine Geometric Detail:** A fixed voxel resolution can fail to capture very thin structures like wires, railings, or chair legs. Addressed by schemes that introduce finer block subdivisions near detailed surfaces, preserving delicate geometry without inflating overall memory usage [S3, unofficial].

## Subtopics

### [Thread-Safe Splitting and Merging](12395-thread-safe-splitting-and-merging.md)
This method manages the dynamic refinement and coarsening of voxel blocks within a thread-safe voxel hash table, enabling parallel updates to sparse volumetric representations like TSDFs. It differs from non-parallel approaches by using atomic compare-and-swap operations on linked list entries to handle concurrent block allocation and deallocation without locks, and from purely static hashing by supporting runtime structural changes. It builds on [Voxel Hashing](2702-voxel-hashing.md) by adding the logic for maintaining consistency during topology changes.

### [Hierarchical Voxel Block Hashing Integration](12411-hierarchical-voxel-block-hashing-integration.md)
Hierarchical Voxel Block Hashing Integration is a method for efficiently integrating depth images into a 3D Truncated Signed Distance Field (TSDF) by using a multi-resolution spatial hash table. It solves the problem of memory inefficiency and cache misses in single-resolution voxel hashing by allocating fine-grained voxel data only where necessary, while maintaining a coarse grid for fast traversal. It sets itself apart from standard Voxel Hashing by introducing a hierarchy of resolution levels, allowing the system to skip empty regions at coarse levels and only allocate memory for active blocks at finer levels, thereby reducing memory footprint and improving parallel allocation throughput on GPUs.

## Operating characteristics
**Time Complexity:** Block allocation and lookup are $O(1)$ on average due to hashing [S3, unofficial]. Fusion is linear in the number of projected depth points per block [S3, unofficial]. Mesh extraction is linear in the number of allocated blocks [S3, unofficial].
**Memory Complexity:** Memory usage is proportional to the surface area of the observed geometry, not the volume of the scene [S3, unofficial].
**Measured Performance:** A well-optimized system can sustain 30 frames per second or more on current graphics hardware [S3, unofficial]. The InfiniTAM implementation exceeds 1000 fps on a high-end NVIDIA GPU [S3, unofficial]. Typical resolutions are 5 mm for human sequences and 1 cm for larger scenes [S2].
**Strengths:** Constant-time access, low memory footprint for sparse scenes, high GPU parallelism [S3, unofficial].
**Limitations:** Hash collisions can cause thread stalls if the load factor is high [S3, unofficial]. Stale blocks require garbage collection [S3, unofficial]. Fixed resolution may miss fine details [S3, unofficial]. Moving objects cause smearing unless dynamic extensions are used [S3, unofficial].
**When to use:** Real-time reconstruction of large, sparse environments where memory is constrained [S3, unofficial].
**When not to use:** Scenes with extremely dense, uniform geometry where a dense grid is more efficient, or when multi-resolution flexibility is critical without the overhead of hierarchical structures [S3, unofficial].

## Background
Voxel Hashing was introduced in the 2013 paper “Real time 3D Reconstruction at Scale Using Voxel Hashing” by Matthias Nießner, Michael Zollhöfer, Shahram Izadi, and Marc Stamminger, published in ACM Transactions on Graphics and presented at SIGGRAPH Asia 2013 [S3, unofficial]. It addressed the limitations of Microsoft’s KinectFusion, which was confined to a ~3-meter cube due to fixed grid allocation [S3, unofficial]. The technique gained widespread attention for enabling affordable depth cameras to produce dense, detailed 3D models of entire rooms on a single consumer GPU [S3, unofficial]. It is used in AR headsets for persistent spatial maps, robotic navigation/SLAM (TU Munich, Stanford), and medical volumetric imaging [S3, unofficial]. It also serves as a backend for SLAM frameworks like ORB SLAM, ElasticFusion, BundleFusion, and NICE SLAM [S3, unofficial]. NVIDIA’s Instant NGP (2022) adapted the concept using a multiresolution hash table of trainable feature vectors [S3, unofficial].

## Sources
[S1] A Comparative Study about Data Structures Used for Efficient Management of Voxelised Full-Waveform Airborne LiDAR Data during 3D Polygonal Model Creation, https://doi.org/10.3390/rs13040559, fetched 2026-10-26
[S2] Implicit Surface Compression -- with Good Old Discrete Cosine Transform and Motion Compensation, https://doi.org/10.1145/3811298, fetched 2026-10-30
[S3] Voxel Hashing: The Definitive Guide to Scalable Real Time 3D ..., https://techyways.com/voxel-hashing/, fetched 2026-10-29 (unofficial)
