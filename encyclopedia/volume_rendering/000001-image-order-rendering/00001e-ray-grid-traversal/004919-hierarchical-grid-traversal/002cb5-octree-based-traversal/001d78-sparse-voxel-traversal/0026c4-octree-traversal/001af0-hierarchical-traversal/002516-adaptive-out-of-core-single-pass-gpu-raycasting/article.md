# Adaptive Out-of-Core Single-Pass GPU Raycasting

Also called: Adaptive out-of-core technique, Single-pass GPU ray casting framework

## Overview
This method enables interactive rendering of multi-gigabyte scalar volumes on desktop hardware by combining an out-of-core octree data structure with a single-pass, stackless GPU raycasting algorithm [S1]. It solves the memory capacity and bandwidth bottlenecks of in-core rendering by maintaining a view- and transfer-function-dependent working set of volumetric bricks in GPU memory, while the rest of the dataset remains on disk [S1]. Unlike separate block rendering techniques that suffer from boundary artifacts and cannot support refractive or global illumination models, this approach performs the entire volume traversal and compositing in a single fragment shader pass, adapting sampling density to the local brick resolution [S1].

## How it works

### Inputs and Outputs
*   **Input:** A massive scalar volume dataset (e.g., 2 Gvoxels, 16-bit) stored on disk.
*   **Output:** A rendered 2D image (e.g., 1024 × 1024 pixels) with accumulated color and opacity, updating the frame buffer and depth buffer [S1].

### Data Structures

#### 1. Out-of-Core Octree
The dataset is decomposed into small cubical bricks and organized into a coarse octree structure maintained out-of-core [S1].
*   **Leaves:** Store the original voxel data for the brick.
*   **Inner Nodes:** Store a filtered representation of their children (aggregated data) [S1].
*   **Node Metadata:** Each node stores:
    *   The range of scalar values present in the node [S1].
    *   Optionally, precomputed gradients [S1].
*   **Boundary Replication:** To support neighbor access for interpolation and gradients, bricks are made self-contained:
    *   One layer of neighboring samples is replicated for linear interpolation support [S1].
    *   Two layers of neighboring samples are replicated if central differences are used for gradient computation at render time [S1].

#### 2. Working Set (GPU Memory)
An adaptive loader on the CPU maintains a subset of bricks in GPU memory. This set is view-dependent and transfer-function-dependent [S1].
*   **Content:** Only the bricks required for the current view and transfer function are loaded.
*   **Management:** Maintained via an adaptive refinement method guided by node priorities [S1].

#### 3. Compact Indexing Structure
At each frame, a compact indexing structure is encoded in a small GPU texture [S1].
*   **Purpose:** Spatially organizes the current working set into an octree hierarchy for the raycaster [S1].
*   **Nature:** It is *not* a multiresolution data representation; it is a spatial index [S1].
*   **Layout:**
    *   **Inner Nodes:** Contain pointers to children [S1].
    *   **Leaves:** Refer to volume data nodes stored in the GPU memory pool (the working set) [S1].
    *   **Pointers:** Include neighbor pointers to facilitate on-the-fly bounding box computation and neighbor access [S1].

### Algorithm Steps

1.  **Preprocessing (Offline):**
    1.  Decompose the original volumetric model into small cubical bricks [S1].
    2.  Organize bricks into a coarse octree structure maintained out-of-core [S1].
    3.  Store original data at leaves and filtered representations at inner nodes [S1].
    4.  Store value ranges and optional precomputed gradients for each node [S1].
    5.  Replicate boundary samples (1 layer for linear interpolation, 2 layers for central difference gradients) to make bricks self-contained [S1].

2.  **Adaptive Loading (CPU/GPU Interaction):**
    1.  Initialize a working set of bricks in GPU memory [S1].
    2.  Compute node priorities based on view and transfer function [S1].
    3.  Asynchronously fetch data from the out-of-core octree to update the working set [S1].
    4.  Refine the working set incrementally based on priorities [S1].
    5.  Receive feedback from the renderer regarding occlusion to avoid loading/refining occluded zones [S1].

3.  **Indexing (Per Frame):**
    1.  Encode the spatial organization of the current working set into a compact texture [S1].
    2.  Populate inner nodes with pointers to children and leaves with references to GPU memory locations [S1].

4.  **Single-Pass Raycasting (GPU Fragment Shader):**
    1.  For each pixel, cast a ray into the volume [S1].
    2.  Traverse the compact indexing structure (stackless octree traversal) [S1].
    3.  Enumerate non-empty bricks in front-to-back order [S1].
    4.  Adapt sampling density to the resolution of the current brick [S1].
    5.  Compute the volume rendering integral (accumulating color and opacity) [S1].
    6.  Stop raycasting when accumulated opacity exceeds a threshold [S1].
    7.  Update the frame buffer and depth buffer [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocessing Phase (CPU/Offline)
def preprocess_volume(dataset):
    # 1. Decompose into bricks
    bricks = decompose_into_cubical_bricks(dataset)
    
    # 2. Build out-of-core octree
    octree = build_octree(bricks)
    
    # 3. Populate node data
    for node in octree.traverse():
        if node.is_leaf():
            node.data = original_voxel_data
        else:
            node.data = filter_children_data(node.children)
        
        node.value_range = compute_value_range(node.data)
        node.gradients = compute_precomputed_gradients(node.data) # Optional
        
        # 4. Replicate boundaries for self-containment
        if node.is_leaf():
            replicate_neighbors(node.data, layers=1) # For linear interpolation
            # replicate_neighbors(node.data, layers=2) # If central differences needed

# Adaptive Loading Phase (CPU/GPU)
def update_working_set(octree, view_params, transfer_function):
    # 1. Compute priorities for nodes
    priorities = compute_node_priorities(octree, view_params, transfer_function)
    
    # 2. Determine working set based on memory budget and priorities
    working_set_nodes = select_nodes_by_priority(octree, priorities)
    
    # 3. Asynchronously fetch data from disk to GPU
    gpu_memory_pool = allocate_gpu_memory(len(working_set_nodes))
    for node in working_set_nodes:
        gpu_memory_pool[node.id] = fetch_data_from_disk(node)
        
    return gpu_memory_pool, working_set_nodes

# Indexing Phase (Per Frame)
def build_index_texture(working_set_nodes, gpu_memory_pool):
    # Create a compact texture representing the spatial hierarchy of the working set
    index_texture = create_texture()
    
    for node in working_set_nodes:
        if node.has_children_in_working_set:
            # Inner node: pointer to children
            index_texture.set_pointer(node.id, node.children_ids)
        else:
            # Leaf node: reference to GPU memory
            index_texture.set_reference(node.id, gpu_memory_pool[node.id].address)
            
    return index_texture

# Raycasting Phase (GPU Fragment Shader)
def raycast_fragment(ray_origin, ray_direction, index_texture, gpu_memory_pool, transfer_function):
    accumulated_color = (0.0, 0.0, 0.0, 0.0)
    accumulated_opacity = 0.0
    threshold = 0.95 # Example threshold
    
    # Stackless traversal of the compact octree index
    current_node = index_texture.root
    t = ray_origin
    
    while current_node is not None and accumulated_opacity < threshold:
        # 1. Traverse to next non-empty brick
        # Compute bounding box on-the-fly using the index structure
        brick_bounds = compute_bounding_box(current_node, index_texture)
        
        # Intersect ray with brick bounds
        t_enter, t_exit = intersect_ray_brick(ray_origin, ray_direction, brick_bounds)
        
        if t_enter is None:
            # Ray missed this brick, move to next sibling or parent
            current_node = next_node_in_traversal(current_node, index_texture)
            continue
            
        t = max(t, t_enter)
        
        # 2. Sample within the brick
        # Adapt sampling density to brick resolution
        step_size = compute_step_size(brick_bounds)
        
        while t < t_exit and accumulated_opacity < threshold:
            sample_pos = ray_origin + ray_direction * t
            
            # 3. Retrieve voxel data
            # Access GPU memory pool using the leaf reference
            voxel_value = sample_voxel(sample_pos, gpu_memory_pool[current_node.ref])
            
            # 4. Transfer function
            color, opacity = apply_transfer_function(voxel_value, transfer_function)
            
            # 5. Front-to-back compositing
            alpha = opacity * (1.0 - accumulated_opacity)
            accumulated_color += color * alpha
            accumulated_opacity += alpha
            
            t += step_size
            
        # 6. Move to next node in front-to-back order
        current_node = next_node_in_traversal(current_node, index_texture)
        
    return accumulated_color, accumulated_opacity
```

### Parameters and Typical Values
*   **Hardware:** NVIDIA 8800 Ultra graphics board [S1].
*   **Resolution:** 1024 × 1024 window size [S1].
*   **Dataset Size:** 2 Gvoxels, 16-bit [S1].
*   **Performance:** Interactive exploration achieved on a desktop PC [S1].

### Edge Cases and Pitfalls
*   **Block Boundaries:** Unlike separate block rendering, this method avoids artifacts at block boundaries because it performs a full-volume traversal [S1].
*   **Occlusion:** Feedback from the renderer is used to avoid refinement and data loading of occluded zones, preventing unnecessary GPU stalls [S1].
*   **Neighbor Access:** Failure to replicate boundary samples correctly will break linear interpolation or gradient computation if neighbor voxels are accessed [S1].

## Operating characteristics

*   **Time Complexity:** The traversal is stackless and exploits the regular structure of octrees to reduce costly texture memory accesses by computing bounding boxes on-the-fly [S1]. The adaptive refinement ensures that only relevant parts of the volume are processed [S1].
*   **Memory Complexity:** Memory usage is bounded by the size of the working set in GPU memory, which is dynamically adjusted based on view and transfer function dependencies [S1]. The rest of the data remains out-of-core [S1].
*   **Performance:** Enables interactive exploration of multi-gigabyte datasets (e.g., 2 Gvoxels) on desktop hardware [S1].
*   **Strengths:**
    *   Supports full-volume optical models, including refracting volumes and global illumination, which separate block rendering cannot easily handle [S1].
    *   Adapts sampling density to brick resolution, providing visual quality proportional to data availability [S1].
    *   Reduces GPU stalls via occlusion query feedback [S1].
*   **Limitations:**
    *   Requires a GPU capable of single-pass fragment shader execution for volume traversal [S1].
    *   Performance depends on the efficiency of the adaptive loader and the compact indexing structure construction [S1].
    *   Unlike flat multiresolution blocking, it does not have a constant number of blocks, which may complicate certain types of parallelization or fixed-size memory allocation strategies [S1].

## Background

*   **Origin:** The method was presented in the paper "A single-pass GPU ray casting framework for interactive out-of-core rendering of massive volumetric datasets" by E. Gobbetti, F. Marton, and J. Jallet [S1].
*   **History:** It builds upon earlier work on out-of-core volume octrees [S1] and stackless ray traversal methods for kd-trees [7, 16], extending them to GPUs for volume rendering [S1]. It addresses the limitations of previous GPU volume rendering techniques that were restricted to moderate-sized volumes fitting entirely in texture memory [S1].
*   **Usage:** Primarily used for interactive exploration of multi-gigabyte CT datasets in medical visualization and numerical simulation analysis [S1].

## Sources

[S1] A single-pass GPU ray casting framework for interactive out-of-core rendering of massive volumetric datasets, https://www.crs4.it/vic/data/papers/cgi2008-movr.pdf, fetched 2026-10-29
