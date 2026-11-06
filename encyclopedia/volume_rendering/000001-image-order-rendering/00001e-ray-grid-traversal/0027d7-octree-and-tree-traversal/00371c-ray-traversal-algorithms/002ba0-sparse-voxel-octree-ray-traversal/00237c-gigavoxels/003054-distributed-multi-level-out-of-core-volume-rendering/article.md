# Distributed Multi-Level Out-Of-Core Volume Rendering

Also called: Display-Aware Multi-Resolution Volume Streaming and Ray-Casting, Distributed Image-Order Volume Rendering, Distributed Image-Order Volume Rendering with Multi-Level Cache, Distributed Volume Rendering, Distributed Volume Rendering with Hierarchical Data Management, Octree/Quadtree-based Distributed Rendering, Vol-a-Tile 2

## Overview
Distributed Multi-Level Out-Of-Core Volume Rendering is a distributed GPU volume rendering system that uses a multi-level hierarchy of caches and data streaming to render volumes larger than GPU memory, maintaining a 1:1 pixel-to-voxel ratio for interactive exploration [S1, unofficial][S2]. It differs from its parent [GigaVoxels](3057-gigavoxels.md) by operating on a distributed-memory cluster with explicit data management across CPU and GPU memory, rather than a single-node ray-guided streaming approach. It also differs from Vol-a-Tile [S2] by not replicating data within a region-of-interest on every node, and from Volume Visualizer [S2] by distributing data rather than replicating the entire dataset.

## How it works
**Inputs and Outputs:**
*   **Input:** Raw volume data (e.g., microscopy, CT), user view transformation, target display resolution.
*   **Output:** Rendered image displayed on the node that rendered it [S2].

**Data Structures:**
1.  **Hierarchical Data Structure:** A conventional octree where each level is associated with a different level-of-detail (LOD) representation. Leaf nodes hold original highest resolution data; root node holds coarsest resolution data [S2].
    *   *Flat Data Variant:* If one dimension is substantially smaller than others, four leaf nodes are left empty, producing a quadtree-like structure where data along the short dimension is not subsampled at each level [S2].
2.  **Brick:** A block of voxels. Dimensions are powers of two (e.g., $64 \times 64 \times 256$, $32 \times 32 \times 128$, $128 \times 256 \times 128$, $64 \times 128 \times 128$, $32 \times 64 \times 128$, $256 \times 256 \times 128$, $128 \times 128 \times 128$, $64 \times 64 \times 128$, $32 \times 32 \times 128$) [S2].
3.  **Cache Entry:** Stores brick ID and texture data [S2].
4.  **Virtual Volume Texturing (S1 specific):** Uses a two-level hierarchy of 3D page caches:
    *   Top-level page directory texture.
    *   Page table cache texture.
    *   Physical 3D cache texture.
    *   Each 2nd-level page table entry refers to a small block of $32^3$ physical voxels [S1, unofficial].
    *   Each page directory entry refers to a block of $32^3$ 2nd-level page table entries [S1, unofficial].
    *   Only the top-level page directory is fully resident in GPU memory [S1, unofficial].

**Algorithm Steps:**

1.  **Preprocessing (Offline):**
    *   Convert raw volume data into an octree or quadtree structure [S2].
    *   Distribute preprocessed bricks in a round-robin, level-by-level fashion among cluster nodes, storing them on local disks [S2].
    *   If a level has fewer bricks than nodes, replicate the entire level on every node [S2].
    *   Compute the view-frustum for each node [S2].

2.  **Initialization (Per Frame/Interaction):**
    *   User interactively determines the view-transformation via a separate user-interface [S2].
    *   Display-awareness selects a target resolution (typically 1:1 pixel-to-voxel) and the corresponding octree level [S1, unofficial].
    *   For each node, create a list of visible bricks by performing a breadth-first search algorithm to traverse the octree based on its frustum and current view transformation [S2].
    *   Sort the visible bricks [S2].

3.  **Data Retrieval (Cache Hierarchy):**
    *   When a brick is needed, the data management system searches three levels [S2]:
        1.  **Texture-Memory Cache (GPU):** Search texture-memory cache first. If found, move to top of cache and report texture data to renderer. If not present, continue search at next level [S2].
        2.  **Memory System Cache (Local RAM):** Search local RAM of the requesting node. If found, move to top of cache and insert into texture-memory cache. If not, query distributed shared-memory [S2].
        3.  **Distributed Shared-Memory/Disk:** Track bricks in all nodes' memory system caches. If present on another node, transfer via high-speed backplane. If not in any cache, load from local disk, transfer to recipient node, and load into recipient's cache [S2].
    *   Both cache levels employ the least-recently-used (LRU) replacement strategy [S2].
    *   Cache size is determined by the allotted memory (set by the user) divided by the brick size [S2].

4.  **Rendering:**
    *   Use hardware accelerated 3D texture-mapping to render the visible list of bricks [S2].
    *   Each brick is rendered as its own individual texture [S2].
    *   If data bricks cross tile boundaries between display nodes, copy the bricks to each node where they are visible [S2].
    *   Swap buffers when reaching the end of each level-of-detail [S2].
    *   *Opacity Correction:* The opacity assigned at each sample point must be corrected to account for the number of samples taken and the current octree level, especially for large, shallow volumes viewed from the side [S2].

5.  **Level-of-Detail (LOD) Management:**
    *   Automatically switch to the coarsest resolution data (lowest LOD) when the user interacts to provide instant feedback [S2].
    *   When navigation stops, progressively render higher resolution data from successive octree levels [S2].
    *   The lowest LOD is always the size of one brick and renders at interactive rates [S2].
    *   While streaming in higher-resolution data, lower-resolution data can be freely mixed in by the ray-caster [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global Constants
BRICK_DIMENSIONS = [
    (64, 64, 256), (32, 32, 128), (128, 256, 128), (64, 128, 128),
    (32, 64, 128), (256, 256, 128), (128, 128, 128), (64, 64, 128),
    (32, 32, 128)
]
NETWORK_MTU = 9000

class Brick:
    def __init__(self, brick_id, data=None):
        self.brick_id = brick_id
        self.data = data # Texture data or raw voxel data

class Cache:
    def __init__(self, capacity_in_bricks):
        self.capacity = capacity_in_bricks
        self.bricks = {} # Map brick_id to Brick
        self.access_order = [] # List of brick_ids for LRU

    def get(self, brick_id):
        if brick_id in self.bricks:
            self.access_order.remove(brick_id)
            self.access_order.append(brick_id)
            return self.bricks[brick_id]
        return None

    def put(self, brick):
        if brick.brick_id in self.bricks:
            self.access_order.remove(brick.brick_id)
        else:
            if len(self.bricks) >= self.capacity:
                lru_id = self.access_order.pop(0)
                del self.bricks[lru_id]
        self.bricks[brick.brick_id] = brick
        self.access_order.append(brick.brick_id)

class DistributedRenderer:
    def __init__(self, cluster_nodes, local_disk_path, texture_memory_size, ram_size, brick_size_bytes):
        self.cluster_nodes = cluster_nodes
        self.local_disk_path = local_disk_path
        self.texture_memory_cache = Cache(capacity_in_bricks=texture_memory_size // brick_size_bytes)
        self.ram_cache = Cache(capacity_in_bricks=ram_size // brick_size_bytes)
        self.distributed_shared_memory = {} # Map brick_id to node_id

    def load_brick_from_disk(self, brick_id):
        # Load from local disk on the node where brick resides
        # Return raw voxel data
        pass

    def transfer_brick(self, brick_id, source_node_id, target_node_id):
        # Transfer via high-speed backplane (MTU 9000)
        pass

    def get_brick(self, brick_id, requesting_node_id):
        # Level 1: Texture-Memory Cache (GPU)
        brick = self.texture_memory_cache.get(brick_id)
        if brick:
            return brick

        # Level 2: Memory System Cache (Local RAM)
        brick = self.ram_cache.get(brick_id)
        if brick:
            # Insert into texture-memory cache
            self.texture_memory_cache.put(brick)
            return brick

        # Level 3: Distributed Shared-Memory / Disk
        source_node_id = self.distributed_shared_memory.get(brick_id)
        if source_node_id is not None:
            # Transfer from another node
            raw_data = self.transfer_brick(brick_id, source_node_id, requesting_node_id)
            brick = Brick(brick_id, raw_data)
            self.ram_cache.put(brick)
            self.texture_memory_cache.put(brick)
            return brick
        else:
            # Load from local disk
            raw_data = self.load_brick_from_disk(brick_id)
            brick = Brick(brick_id, raw_data)
            self.ram_cache.put(brick)
            self.texture_memory_cache.put(brick)
            return brick

    def render_frame(self, view_transformation, target_resolution):
        # 1. Visibility Culling
        visible_bricks = []
        for node in self.cluster_nodes:
            # Breadth-first search on octree based on frustum
            bricks = node.traverse_octree(view_transformation)
            visible_bricks.extend(bricks)
        
        # 2. LOD Management
        # If user interacting, use coarsest LOD. Else, progressive LOD.
        lod_level = self.select_lod(view_transformation)
        
        # 3. Rendering
        for brick_id in visible_bricks:
            # Ensure brick is in texture memory
            brick = self.get_brick(brick_id, self.current_node_id)
            
            # Handle tile boundary crossing
            if brick.crosses_tile_boundary():
                self.copy_brick_to_other_nodes(brick)
            
            # Render brick using hardware accelerated 3D texture-mapping
            # Correct opacity for sample count and octree level
            self.render_brick(brick, lod_level)
```

## Operating characteristics
*   **Time Complexity:** Not explicitly given in the sources. Performance scales with output resolution and cluster size [S2].
*   **Memory Complexity:** Bounded by the working set size required for ray-casting, which is display-aware [S1, unofficial]. The total working set memory can be increased by using multiple GPU nodes [S1, unofficial].
*   **Measured Performance:**
    *   **Purkinje Neuron Dataset:** 2,048 x 4,096 x 128 volume, 16-bit voxels, 2 GB original size. Rendered on 3-node cluster (AMD Athlon 64 FX-60, 2 GB RAM, GeForce 7900 GT) with 24-megapixel display. Best brick size: 64x128x128. Render times: 3.4s (8 nodes/8 MP), 1.7s (16 nodes/16 MP), 0.5s (24 nodes/24 MP) [S2].
    *   **Rat Kidney Dataset:** 32,768 x 32,768 x 128 volume, 24-bit samples, 384 GB original size. Rendered on 28-node cluster (AMD Opteron 246, 4 GB RAM, Quadro FX 3000) with 100-megapixel display. Best brick size: 64x64x128. Render times: 2625s (15 nodes/92 MP), 1100s (30 nodes/100 MP), 525s (60 nodes/100 MP), 230s (25 nodes/100 MP), 198s (28 nodes/100 MP) [S2].
    *   **Visible Human Female CT:** 512 x 512 x 2,048 volume (padded from 1,871), 16-bit. Rendered on 3-node cluster with 24-megapixel display. Best brick size: 64x64x256. Render times: 2.1s (8 nodes/8 MP), 0.6s (16 nodes/16 MP), 0.3s (24 nodes/24 MP) [S2].
    *   **EM Volumes (S1):** 43 GB and 92 GB volumes rendered on 1 to 8 Tesla nodes. Render times for 43 GB volume on 1 node: 1.28 sec, 0.34 sec, 0.27 sec, 0.70 sec, 0.28 sec, 0.18 sec, 0.19 sec, 0.15 sec, 0.12 sec, 0.7 sec, 0.33 sec, 0.34 sec [S1, unofficial].
*   **Scaling:** The system scales as output resolution and cluster size increase [S2]. Fewer nodes can render the entire volume but with reduced resolution; more nodes increase the working set allowing full resolution rendering [S1, unofficial].
*   **Strengths:** Interactive exploration of large volumes; scalable to high-resolution displays; efficient memory usage via multi-level caching.
*   **Limitations:** Requires distributed-memory cluster; preprocessing time for octree/quadtree conversion; opacity artifacts if not corrected for LOD changes.

## Background
*   **Origin:** The method was presented in "Distributed Volume Rendering for Scalable High-resolution Display Arrays" (GRAPP 2010) [S2] and "Distributed Multi-Level Out-Of-Core Volume Rendering" (NVIDIA GTC 2010 poster) [S1, unofficial].
*   **History:** It builds on previous work like Vol-a-Tile [S2] and Volume Visualizer [S2], addressing their limitations of data replication. It was implemented in C++ using OpenGL, Cg, and MPICH libraries, tested on openSUSE 10.2 Linux [S2].
*   **Usage:** Used in Vol-a-Tile 2 to visualize large microscopy datasets on scalable high-resolution LCD display arrays [S2]. Also used for interactive exploration of neuroscience EM volumes, specifically mouse hippocampus data from Harvard Center for Brain Science [S1, unofficial]. Implemented on Tesla GPU nodes [S1, unofficial].

## Sources

[S1] Distributed Multi-Level Out-Of-Core Volume Rendering, https://www.nvidia.com/content/GTC/posters/2010/O01-Distributed-Multi-Level-Out-of-Core-Volume-Rendering.pdf, fetched 2026-10-26 (unofficial)
[S2] Distributed Volume Rendering for Scalable High-resolution Display Arrays, https://www.evl.uic.edu/documents/grapp_2010_8_cr.pdf, fetched 2026-10-27
