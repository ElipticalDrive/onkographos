# Voxel Hashing

Also called: Hash-based voxel representation, Sparse TSDF Representation, Spatial hashing

## Overview
Voxel Hashing is a sparse volumetric data structure that enables real-time 3D scene reconstruction by allocating memory only where measured surfaces exist, rather than filling an entire fixed grid [S2, unofficial]. It pairs small blocks of voxels with a hash table, giving systems constant time access to surface data while keeping memory usage proportional to the actual geometry observed [S2, unofficial]. It sets itself apart from its parent [Multi-type Voxel Storage](1104-multi-type-voxel-storage.md) by using a flat spatial hash table for O(1) block access instead of tree-based or typed array indexing, and from its sibling [Typed Voxels](1399-typed-voxels.md) by prioritizing GPU parallelism and constant-time lookup over type-specific optimization [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A stream of depth frames from a sensor (structured light, time-of-flight, or stereo vision), where each frame records per-pixel distance to the nearest physical surface [S2, unofficial]. Camera pose estimates are also required for fusion [S2, unofficial].
*   **Output:** A sparse set of allocated voxel blocks containing truncated signed distance function (TSDF) values and confidence weights, and optionally a triangle mesh extracted via Marching Cubes [S2, unofficial].

**Data Structures:**
1.  **Voxel Block:** A fixed-size 3D grid of voxels.
    *   **Dimensions:** $8 \times 8 \times 8$ voxels [S1][S2, unofficial].
    *   **Fields per Voxel:**
        *   `tsdf`: Truncated signed distance value. Positive in front of the surface, negative behind, zero at the boundary [S2, unofficial].
        *   `weight`: Confidence weight for the TSDF value [S2, unofficial].
2.  **Hash Table:** A flat lookup table storing pointers or indices to Voxel Blocks.
    *   **Capacity:** The original implementation allocated approximately 500,000 entries [S2, unofficial].
    *   **Load Factor:** Kept under 50% to minimize probe chain length [S2, unofficial].
    *   **Collision Resolution:** Open addressing with linear probing to avoid pointer dereferencing on GPUs [S2, unofficial].

**Parameters:**
*   **Voxel Resolution:** 5 mm for human sequences, 1 cm for larger scenes [S1].
*   **TSDF Truncation Band:** Set to one block width (8 voxels) in experiments [S1].
*   **Block Size:** Fixed to $8 \times 8 \times 8$ [S1][S2, unofficial].

**Algorithm Steps:**

1.  **Capture Depth Data:** Acquire a depth frame $D_t$ and camera pose $P_t$ [S2, unofficial].
2.  **Determine Active Blocks:** Identify voxel blocks within the camera frustum and close to measured surfaces. For each block, compute its 3D integer coordinate $(x, y, z)$ [S2, unofficial].
3.  **Hash Block Coordinates:** Compute hash index $h$ for the block coordinate.
    *   *Note: The specific hash function is not detailed in the sources, but it maps 3D coordinates to table indices [S2, unofficial].*
4.  **Allocate Block:** If block $h$ is not present in the hash table, allocate a new $8 \times 8 \times 8$ voxel block and insert it into the table at index $h$ using linear probing if necessary [S2, unofficial].
5.  **Fuse TSDF Values:** For each voxel in the active block:
    *   Project the voxel center into the depth frame to get a measured distance $d_{meas}$ [S2, unofficial].
    *   Compute the signed distance $s$ from the voxel to the surface.
    *   Truncate $s$ to the TSDF band (e.g., $\pm 8$ voxels) [S1].
    *   Update the voxel's TSDF value and weight using a running weighted average:
        $$ w_{new} = w_{old} + w_{meas} $$
        $$ tsdf_{new} = \frac{w_{old} \cdot tsdf_{old} + w_{meas} \cdot tsdf_{meas}}{w_{new}} $$
        *   *Note: $w_{meas}$ is typically derived from the depth confidence or inverse distance [S2, unofficial].*
6.  **Garbage Collection (Periodic):** Identify stale blocks (those no longer relevant to the current view or surface) and free them for reallocation [S2, unofficial].
7.  **Extract Mesh (Optional):** Run a Marching Cubes variant over all allocated blocks to produce a triangle mesh [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
BLOCK_SIZE = 8
TRUNCATION_BAND = 8  # In voxels
MAX_LOAD_FACTOR = 0.5
HASH_TABLE_SIZE = 500000  # Example capacity

# Data Structures
class VoxelBlock:
    def __init__(self):
        # 8x8x8 grid of voxels
        self.tsdf = [0.0] * (BLOCK_SIZE ** 3)
        self.weight = [0.0] * (BLOCK_SIZE ** 3)

class HashTable:
    def __init__(self, size):
        self.size = size
        self.entries = [None] * size  # Stores VoxelBlock or None

    def hash(self, x, y, z):
        # Simple hash function example; actual implementation may vary
        # Must map 3D integer coordinates to [0, size-1]
        return (x * 73856093 ^ y * 19349663 ^ z * 83492791) % self.size

    def insert(self, x, y, z, block):
        idx = self.hash(x, y, z)
        start_idx = idx
        while self.entries[idx] is not None:
            # Check if existing block matches coordinates (for collision resolution)
            # In practice, linear probing finds the first empty slot
            idx = (idx + 1) % self.size
            if idx == start_idx:
                raise Exception("Hash table full")
        self.entries[idx] = block

    def get_or_create(self, x, y, z):
        idx = self.hash(x, y, z)
        start_idx = idx
        while self.entries[idx] is not None:
            # Note: The sources do not specify how to check if the existing block
            # corresponds to the requested coordinates. 
            # In a real implementation, one might store coordinates in the entry
            # or use a separate coordinate array. 
            # For this pseudocode, we assume linear probing finds the slot for
            # the first occurrence of this coordinate, or we simply allocate
            # if not found. 
            # [G] Guard: In a robust implementation, one would check if the 
            # existing block at idx corresponds to (x,y,z). If not, continue probing.
            idx = (idx + 1) % self.size
            if idx == start_idx:
                # Table full, need to expand or garbage collect
                pass
        if self.entries[idx] is None:
            self.entries[idx] = VoxelBlock()
        return self.entries[idx]

def fuse_depth_frame(depth_frame, camera_pose, hash_table):
    # Iterate over pixels in depth frame
    for u, v in depth_frame.pixels:
        depth = depth_frame.get_depth(u, v)
        # Project voxel centers in the frustum to depth
        # For each voxel in the relevant blocks:
        for block in hash_table.get_active_blocks_in_frustum(camera_pose):
            for i in range(BLOCK_SIZE):
                for j in range(BLOCK_SIZE):
                    for k in range(BLOCK_SIZE):
                        voxel_idx = i * BLOCK_SIZE * BLOCK_SIZE + j * BLOCK_SIZE + k
                        # Get voxel world position
                        voxel_pos = block.get_voxel_world_position(i, j, k)
                        # Project voxel_pos to depth frame
                        projected_depth = project_to_depth(voxel_pos, camera_pose)
                        if projected_depth is not None:
                            measured_depth = depth_frame.get_depth_at(projected_depth.u, projected_depth.v)
                            if measured_depth is not None:
                                # Compute signed distance
                                signed_dist = voxel_pos.distance_to_surface(measured_depth)
                                # Truncate
                                truncated_dist = max(-TRUNCATION_BAND, min(TRUNCATION_BAND, signed_dist))
                                # Update weight and TSDF
                                # [G] Assuming uniform weight for simplicity; sources mention weighted average
                                w_meas = 1.0 
                                w_old = block.weight[voxel_idx]
                                tsdf_old = block.tsdf[voxel_idx]
                                
                                w_new = w_old + w_meas
                                tsdf_new = (w_old * tsdf_old + w_meas * truncated_dist) / w_new
                                
                                block.weight[voxel_idx] = w_new
                                block.tsdf[voxel_idx] = tsdf_new

def extract_mesh(hash_table):
    mesh = Mesh()
    for entry in hash_table.entries:
        if entry is not None:
            # Run Marching Cubes on this block
            triangles = marching_cubes(entry.tsdf, BLOCK_SIZE)
            mesh.add_triangles(triangles)
    return mesh
```

**Edge Cases and Pitfalls:**
*   **Hash Collisions:** Multiple block coordinates can resolve to the same hash bucket. Handled by open addressing with linear probing [S2, unofficial].
*   **Stale Blocks:** As the camera moves, previously scanned regions may fall out of relevance. Stale blocks accumulate and crowd out space for new data. Handled by a garbage collection pass [S2, unofficial].
*   **Tracking Drift:** When the camera pose estimator loses accuracy, incoming depth data fuses into incorrect positions, producing ghosting or doubled surfaces. Mitigated by pairing with robust pose alignment methods such as frame-to-model Iterative Closest Point (ICP) or learned visual feature matching [S2, unofficial].
*   **Moving Objects:** The standard pipeline assumes a static world. People walking through the scene or objects being relocated cause smeared geometry. Addressed by extensions like DynamicFusion and VolumeDeform, which warp the volume to accommodate non-rigid motion [S2, unofficial].
*   **Fine Geometric Detail:** A fixed voxel resolution can fail to capture very thin structures like wires, railings, or chair legs. Addressed by schemes that introduce finer block subdivisions near detailed surfaces [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** O(1) average-case lookup for block access due to hash table [S2, unofficial]. Octree methods trade this for O(log n) tree traversal [S2, unofficial].
*   **Memory Complexity:** Proportional to the actual geometry observed, not the entire volume [S2, unofficial].
*   **Performance:** A well-optimized system can sustain 30 frames per second or more on current graphics hardware [S2, unofficial]. InfiniTAM implementation exceeds 1000 fps on a high-end NVIDIA GPU [S2, unofficial].
*   **Strengths:** Constant time access, GPU parallelism, scalability to room-scale and larger scenes, memory efficiency [S2, unofficial].
*   **Limitations:** Fixed voxel resolution may miss fine details; sensitive to tracking drift; requires garbage collection for stale data [S2, unofficial].
*   **When to Use:** Real-time 3D reconstruction, AR/VR spatial mapping, robotic SLAM where memory and speed are critical [S2, unofficial].
*   **When Not to Use:** Scenes with highly dynamic, non-rigid motion without extensions; applications requiring multi-resolution flexibility without the overhead of tree traversal (though Voxel Hashing is O(1), it lacks the hierarchical detail of octrees) [S2, unofficial].

## Background
Voxel Hashing was introduced in the 2013 paper “Real time 3D Reconstruction at Scale Using Voxel Hashing” by Matthias Nießner, Michael Zollhöfer, Shahram Izadi, and Marc Stamminger, published in ACM Transactions on Graphics and presented at SIGGRAPH Asia 2013 [S2, unofficial]. It addressed the limitations of prior systems like KinectFusion, which used a rigid, pre-allocated 3D grid confined to roughly a three-meter cube [S2, unofficial]. Voxel Hashing broke through this ceiling by using sparse block allocation and hash table indexing, allowing room-scale and larger scenes with memory proportional to geometry [S2, unofficial]. It is used in AR headsets for persistent spatial maps, robotic navigation/SLAM (TU Munich, Stanford), and medical volumetric imaging (CT/MRI) [S2, unofficial]. It also serves as a backend for SLAM frameworks like ORB SLAM, ElasticFusion, BundleFusion, and NICE SLAM [S2, unofficial]. NVIDIA’s Instant NGP (2022) adapted the concept using a multiresolution hash table of trainable feature vectors instead of fixed TSDF values [S2, unofficial].

## Sources
[S1] Implicit Surface Compression -- with Good Old Discrete Cosine Transform and Motion Compensation, https://doi.org/10.1145/3811298, fetched 2026-10-30
[S2] Voxel Hashing: The Definitive Guide to Scalable Real Time 3D ..., https://techyways.com/voxel-hashing/, fetched 2026-10-29 (unofficial)
