# Distributed Multi-Level Out-Of-Core Volume Rendering

Also called: Display-Aware Multi-Resolution Volume Streaming and Ray-Casting, Distributed Image-Order Volume Rendering, Distributed Image-Order Volume Rendering with Multi-Level Cache, Distributed Volume Rendering, Distributed Volume Rendering with Hierarchical Data Management, Octree/Quadtree-based Distributed Rendering, Vol-a-Tile 2

## Overview
Distributed Multi-Level Out-Of-Core Volume Rendering is a distributed image-order rendering technique designed to visualize volume datasets significantly larger than the aggregate memory of a GPU cluster, targeting scalable high-resolution display arrays [S2]. It solves the problem of out-of-core data access by combining a multi-level cache hierarchy (GPU texture memory, CPU RAM, and distributed shared memory) with a display-aware LOD selection mechanism that bounds the working set size [S1, unofficial][S2]. Unlike sibling methods such as GigaVoxels which stream data to a single GPU, this method distributes the volume across cluster nodes, using a unique data management scheme to relocate bricks between nodes based on view transformation, thereby avoiding the data replication required by systems like Vol-a-Tile or Volume Visualizer [S2].

## How it works
The method operates on a distributed-memory cluster where each node possesses local storage, CPU RAM, and a GPU. The volume is decomposed into a hierarchical structure (octree or quadtree) of "bricks" [S2].

### Inputs and Outputs
*   **Input:** Raw volume data (preprocessed into a hierarchical structure), user-defined view transformation, target display resolution.
*   **Output:** A composite image displayed on a high-resolution display array connected to the cluster [S2].

### Data Structures
1.  **Hierarchical Data Structure:** A conventional octree where each level corresponds to a different level-of-detail (LOD). Leaf nodes hold the highest resolution data; the root holds the coarsest [S2].
    *   *Flat Data Variant:* If one dimension is substantially smaller (e.g., microscopy slices), four leaf nodes are left empty to create a quadtree-like structure, preventing subsampling along the short dimension [S2].
2.  **Brick:** A sub-volume block. Dimensions are powers of two (e.g., $64 \times 64 \times 256$, $32 \times 32 \times 128$) to satisfy GPU texture limits [S2].
3.  **Cache Entries:**
    *   **Texture-Memory Cache (Level 1):** Resides in GPU VRAM. Stores brick ID and texture data [S2].
    *   **Memory System Cache (Level 2):** Resides in CPU RAM. Stores brick ID and raw data [S2].
    *   **Distributed Shared-Memory (DSM):** Tracks brick IDs and their physical location (node ID) across the cluster [S2].
4.  **Virtual Volume Texturing (S1 variant):** Uses a two-level 3D page cache hierarchy:
    *   **Page Directory:** Top-level texture fully resident in GPU memory. Each entry refers to a block of 2nd-level page table entries [S1, unofficial].
    *   **Page Table Cache:** Stores virtual page table entries. Each entry refers to a small block of physical voxels (e.g., $32^3$) [S1, unofficial].
    *   **Physical Cache Texture:** Large 3D texture in GPU RAM holding the actual voxel data [S1, unofficial].

### Preprocessing and Distribution
1.  **Preprocessing:** Convert raw data into an octree/quadtree offline. Compute view frustums for each node [S2].
2.  **Distribution:** Distribute preprocessed bricks across cluster nodes' local disks in a round-robin, level-by-level fashion. If a level has fewer bricks than nodes, replicate the entire level on every node (typically only applies to coarse levels) [S2].

### Rendering Loop (Per Frame)
1.  **Visibility Culling:** Each node independently performs a breadth-first search on the octree using the current view transformation to create a sorted list of visible bricks [S2].
2.  **LOD Selection (Display-Awareness):**
    *   Determine target pixel-to-voxel ratio (typically 1:1) [S1, unofficial].
    *   Select the octree level that matches this resolution.
    *   If the required blocks for this level do not fit into the physical texture cache, downgrade to a coarser octree level [S1, unofficial].
    *   *Interaction Handling:* Automatically switch to the coarsest LOD (lowest detail) during user interaction for instant feedback. Progressively render higher resolutions from successive levels once navigation stops [S2].
3.  **Data Retrieval (Multi-Level Cache Lookup):** For each visible brick, the data management system searches three levels:
    1.  **Texture-Memory Cache:** Search GPU texture memory. If found, move to top of LRU list and report data. If not, proceed to Level 2 [S2].
    2.  **Memory System Cache:** Search local CPU RAM. If found, move to top of LRU list, transfer to GPU, and insert into Texture-Memory Cache. If not, proceed to Level 3 [S2].
    3.  **Distributed Shared-Memory / Disk:** Query DSM for brick location.
        *   If present on another node's Memory System Cache, transfer via high-speed backplane (e.g., Ethernet, MTU 9000 bytes) to the requesting node's RAM, then to GPU [S2].
        *   If not in any cache, load from local disk on the owning node, transfer to requesting node, and load into caches [S2].
4.  **Rendering:**
    *   Render each brick using hardware-accelerated 3D texture mapping. Each brick is treated as an individual texture [S2].
    *   *Boundary Handling:* If a brick crosses tile boundaries between display nodes, copy the brick to each node where it is visible [S2].
    *   *Opacity Correction:* Correct opacity assignment at each sample point to account for the number of samples taken and the current octree level, especially for large, shallow volumes viewed from the side [S2].
    *   *Virtual Texturing (S1 variant):* Compute sample positions in virtual volume space. Fetch physical data from the single large 3D cache texture using the two-level page table hierarchy for address translation [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global State
cluster_nodes = [...] # List of node objects
volume_tree = Octree() # Preprocessed hierarchical structure
brick_cache_gpu = LRU_Cache() # Texture-memory cache (GPU)
brick_cache_cpu = LRU_Cache() # Memory system cache (CPU)
dsm_registry = DistributedSharedMemory() # Tracks brick locations

def render_frame(view_transform, display_resolution):
    # 1. Visibility Culling
    visible_bricks = []
    for node in cluster_nodes:
        # Breadth-first search on volume_tree based on view_transform
        bricks = node.perform_bfs_culling(volume_tree, view_transform)
        visible_bricks.extend(bricks)
    
    # 2. LOD Selection (Display-Aware)
    # Determine target pixel-to-voxel ratio (e.g., 1:1)
    target_ratio = 1.0
    lod_level = select_lod_level(volume_tree, display_resolution, target_ratio)
    
    # Check if LOD fits in GPU cache; if not, use coarser level
    if not fits_in_gpu_cache(visible_bricks, lod_level):
        lod_level = lod_level - 1
        visible_bricks = filter_bricks_by_lod(visible_bricks, lod_level)

    # 3. Data Retrieval and Rendering
    for brick in visible_bricks:
        # Ensure brick is in GPU texture memory
        gpu_texture = ensure_in_gpu_cache(brick)
        
        # Handle tile boundaries: copy brick to nodes where it is visible
        if brick.crosses_tile_boundary():
            distribute_brick_to_nodes(brick)
            
        # Render brick using hardware 3D texture mapping
        render_brick(gpu_texture, view_transform)
        
        # Correct opacity for LOD and sample count
        correct_opacity(brick.lod_level, sample_count)

def ensure_in_gpu_cache(brick):
    # Level 1: Texture-Memory Cache (GPU)
    if brick.id in brick_cache_gpu:
        brick_cache_gpu.move_to_top(brick.id)
        return brick_cache_gpu.get_texture(brick.id)
    
    # Level 2: Memory System Cache (CPU)
    if brick.id in brick_cache_cpu:
        brick_cache_cpu.move_to_top(brick.id)
        data = brick_cache_cpu.get_data(brick.id)
        texture = upload_to_gpu(data)
        brick_cache_gpu.insert(brick.id, texture)
        return texture
    
    # Level 3: Distributed Shared-Memory / Disk
    source_node = dsm_registry.lookup(brick.id)
    if source_node is not None:
        # Transfer from another node's CPU cache via backplane
        data = transfer_from_node(source_node, brick.id)
        brick_cache_cpu.insert(brick.id, data)
        brick_cache_cpu.move_to_top(brick.id)
        texture = upload_to_gpu(data)
        brick_cache_gpu.insert(brick.id, texture)
        return texture
    else:
        # Load from local disk on source node
        data = load_from_disk(source_node, brick.id)
        transfer_to_node(source_node, brick.id, data)
        brick_cache_cpu.insert(brick.id, data)
        brick_cache_cpu.move_to_top(brick.id)
        texture = upload_to_gpu(data)
        brick_cache_gpu.insert(brick.id, texture)
        return texture

def select_lod_level(tree, resolution, ratio):
    # Find the coarsest level where pixel-to-voxel ratio <= target ratio
    # and the working set fits in available memory
    for level in tree.levels:
        if level.matches_resolution(resolution, ratio):
            return level
    return tree.coarsest_level
```

## Subtopics

### [Demand-Driven Volume Rendering](8594-demand-driven-volume-rendering.md)
A GPU ray-casting technique that renders a "virtual octree" where physical voxel data is computed on demand, driven by visibility and screen-projection size, allowing interactive rendering of terascale electron microscopy (EM) data without pre-computing a 3D multi-resolution hierarchy. It differs from sibling methods like GigaVoxels by decoupling the ray-caster from the data reconstruction backend, using a two-level page table hierarchy to translate virtual sample positions to physical data on the fly, enabling interactive frame rates for volumes exceeding GPU memory.

## Operating characteristics
*   **Time Complexity:** Rendering time scales with the number of visible bricks and the cluster size. Performance improves as output resolution and cluster size increase because the number of bricks each node must render decreases [S2].
*   **Memory Complexity:** Bounded by the cache sizes (GPU texture memory + CPU RAM) and the distributed shared-memory registry. The working set size is explicitly bounded by the display-aware LOD selection [S1, unofficial][S2].
*   **Measured Performance:**
    *   *Purkinje Neuron (2 GB, 24 MP):* 3 nodes, 0.5s/frame [S2].
    *   *Rat Kidney (384 GB, 100 MP):* 28 nodes, 198s/frame (lowest LOD) to 2625s/frame (highest LOD) [S2].
    *   *Visible Human CT (1 GB, 24 MP):* 24 nodes, 0.3s/frame [S2].
    *   *EM Volumes (43 GB, 92 GB):* Interactive rates (0.12s - 1.28s) on 1-8 Tesla nodes depending on LOD and node count [S1, unofficial].
*   **Strengths:** Scales to very large datasets (terabytes) and high-resolution displays (100+ MP). Avoids data replication across nodes (unlike Vol-a-Tile). Display-awareness ensures interactive feedback during navigation.
*   **Limitations:** Requires complex data management infrastructure (distributed shared memory, multi-level caches). Brick boundary crossing requires data copying. Opacity artifacts can occur if LOD changes are not handled correctly. Flat volumes require special quadtree handling [S2].
*   **When to Use:** Interactive exploration of massive volumes (e.g., neuroscience EM, microscopy) on high-resolution display arrays connected to distributed clusters.
*   **When Not to Use:** Small volumes fitting in GPU memory (overhead of distribution is unnecessary). Real-time applications requiring strict determinism without network latency.

## Background
The method originated from research into distributed volume rendering for scalable high-resolution display arrays. Key implementations include "Vol-a-Tile 2" and the NVIDIA poster on "Display-Aware Multi-Resolution Volume Streaming and Ray-Casting" [S1, unofficial][S2]. It builds on earlier systems like Vol-a-Tile (Schwarz et al., 2004) and Volume Visualizer (McCorquodale and Lombeyda, 2003), addressing their limitations of data replication [S2]. It is used for interactive exploration of neuroscience datasets, such as mouse hippocampus data from the Harvard Center for Brain Science [S1, unofficial]. Implementations have been tested on clusters with AMD Opteron/Athlon processors and NVIDIA GeForce/Quadro GPUs, using C++, OpenGL, Cg, and MPICH [S2].

## Sources
[S1] PDF poster_blue_horizontal_18 - NVIDIA, https://www.nvidia.com/content/GTC/posters/2010/O01-Distributed-Multi-Level-Out-of-Core-Volume-Rendering.pdf, fetched 2026-10-26 (unofficial)
[S2] PDF Distributed Volume Rendering for Scalable High-resolution Display Arrays, https://www.evl.uic.edu/documents/grapp_2010_8_cr.pdf, fetched 2026-10-27
