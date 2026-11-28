# Ray-Guided Volume Rendering

Also called: Hierarchical grids with bricking accessed via multi-level page tables, Multi-level page table hierarchy, Visualization-driven approaches, Visualization-driven volume rendering

## Overview
Ray-guided volume rendering is a category of volume renderers that incorporate a feedback loop between the ray-caster and the culling mechanism, where the ray-caster itself writes out accurate information on missing bricks and brick usage to determine an accurate working set directly on the GPU [S1]. Unlike traditional approaches where the working set is computed on the CPU prior to rendering, ray-guided renderers build up the current working set during ray traversal on the GPU [S1]. This method sets itself apart from its siblings, such as [Bounding-Sphere Tree Traversal with Frame-Rate Control](7227-bounding-sphere-tree-traversal-with-frame-rate-control.md) and [Frustum and Visibility Culling](12355-frustum-and-visibility-culling.md), by eliminating explicit CPU-side culling steps and instead using the ray traversal process itself to implicitly determine visibility and data requirements [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A hierarchical volume data structure (either an octree or a hierarchical grid with bricking) and a transfer function.
*   **Output:** A rendered image and a list of required bricks for the current frame.

**Data Structures:**
The method relies on one of two primary data structures, accessed via a multi-level page table hierarchy or a tree structure with a node/brick pool [S1]:
1.  **Octree/Tree Structure:**
    *   **Node Pool:** Stores active tree nodes. Each node contains pointers to child nodes in the pool and a pointer to an associated texture brick in the brick pool [S1].
    *   **Brick Pool:** A shared cache texture storing the actual voxel data (bricks) [S2, unofficial].
2.  **Hierarchical Grid with Multi-level Page Tables:**
    *   **Page Tables:** Multi-level structures mapping virtual addresses to physical brick locations. Entries are flagged as mapped or unmapped [S1].
    *   **Brick Pool:** Shared cache texture for voxel data [S1].

**Numbered Steps:**
1.  **Ray Traversal:** Rays traverse the volume structure (e.g., using the kd-restart algorithm for octrees [S1]) from front to back.
2.  **Working Set Determination:** As rays traverse, they determine the required data resolution automatically, either on a per-sample basis [HBJP12] or per-brick basis [FSK13] [S1].
3.  **Missing Brick Detection:** The ray-caster detects if a required brick is missing from GPU memory (e.g., a missing octree node, an unmapped page table entry, or a "missing" subtree flag) [S1].
4.  **Write-out:** The ray-caster writes out accurate information on missing bricks and brick usage. This is typically done using multiple render targets or modern OpenGL extensions such as `GL_ARB_shader_storage_buffer_object` [S1][S2, unofficial].
5.  **Implicit Culling:** Visibility is determined during ray traversal, providing implicit view frustum culling without extra steps [S2, unofficial]. Rays also determine implicit occlusion culling without extra steps or occlusion buffers [S2, unofficial].
6.  **Data Streaming:** The written-out information is used to load missing data into the GPU cache [S1].
7.  **Cache Management:** The usage information is used to determine which bricks can be evicted from the GPU cache because they are no longer needed [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for a single ray's interaction with the ray-guided system
# Assumes a global 'BrickPool' and 'PageTable' or 'NodePool'

def ray_guided_traverse(ray_origin, ray_direction, max_depth):
    # Initialize traversal state
    current_node = root_node
    t = 0.0
    
    # List to store bricks required by this ray
    required_bricks = []
    
    while current_node is not None and t < max_distance:
        # Determine if the current node/brick is in GPU memory
        is_present = check_gpu_presence(current_node)
        
        if not is_present:
            # Write out request for missing brick
            # This writes to a buffer accessible by the CPU or a streaming manager
            write_missing_brick_request(current_node.brick_id)
            
            # For the purpose of rendering this frame, we might skip or use lower res
            # [G] Implementation detail: Skip rendering or substitute with lower resolution
            current_node = get_next_node_in_traversal(current_node)
            continue
            
        # If present, sample the brick
        sample = sample_brick(current_node.brick_id, ray_origin, ray_direction, t)
        
        # Accumulate color/opacity based on transfer function
        accumulate_render_output(sample)
        
        # Record this brick as used for cache management
        record_brick_usage(current_node.brick_id)
        
        # Advance ray
        t = advance_ray_t(ray_origin, ray_direction, current_node, t)
        current_node = get_next_node_in_traversal(current_node)
        
    return required_bricks

def check_gpu_presence(node):
    # For octree: check if node pointer is valid in NodePool
    # For page tables: check if page table entry is flagged as mapped
    if node is None:
        return False
    if node.brick_id is None:
        return False
    return True

def write_missing_brick_request(brick_id):
    # Writes to a Shader Storage Buffer Object or Render Target
    # [G] Implementation detail: Append to a global atomic counter buffer
    append_to_missing_brick_buffer(brick_id)

def record_brick_usage(brick_id):
    # Writes to a usage buffer to inform eviction policy
    # [G] Implementation detail: Increment usage counter or set timestamp
    append_to_usage_buffer(brick_id)
```

**Parameters and Typical Values:**
*   **Traversal Algorithm:** kd-restart algorithm is commonly used for octree traversal [S1].
*   **Resolution Determination:** Can be per-sample [HBJP12] or per-brick [FSK13].
*   **OpenGL Extensions:** `GL_ARB_shader_storage_buffer_object` is used for writing out working set information [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Missing Bricks:** The renderer must handle missing bricks gracefully, either by skipping them or substituting lower resolution data [S2, unofficial].
*   **Working Set Too Large:** If the working set exceeds GPU memory, strategies include switching from single-pass to multi-pass rendering or interrupting rendering [S2, unofficial].
*   **Precision:** Not given in the sources.

## Subtopics

### Gigavoxels System
Introduced by Crassin et al. in 2009, this system uses GPU-based octree volume rendering with ray-guided streaming. It employs an N^3 tree (octree) and the kd-restart algorithm for traversal, storing active nodes in a node pool and voxel data in a brick pool [S1]. It targets entertainment applications and supports real-time transfer function updates [S1].

### Multi-level Page Table Hierarchy
Proposed by Hadwiger et al. in 2012, this approach uses hierarchical grids with bricking accessed via multi-level page tables instead of a tree structure. It scales to petavoxel data and efficiently handles highly anisotropic data common in electron microscopy [S1]. It offers advantages in traversal complexity and cache access behavior compared to standard octree traversal for very large data [S1].

## Operating characteristics
*   **Time and Memory Complexity:** Scales to petavoxel data [S1]. The approach illustrates advantages in terms of traversal complexity and cache access behavior compared to standard octree traversal for very large data [S1].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales to petavoxel data [S1].
*   **Strengths:**
    *   Efficiently handles highly anisotropic data, common in high-resolution electron microscopy volumes [S1].
    *   Provides implicit view frustum and occlusion culling without extra steps or buffers [S2, unofficial].
    *   Determines the working set directly on the GPU, reducing CPU-GPU synchronization overhead [S1].
    *   Automatically determines locally required data resolution [S1].
*   **Limitations:**
    *   Requires handling missing bricks and potential cache eviction policies [S1].
    *   If the working set is too large, rendering may need to be interrupted or switched to multi-pass [S2, unofficial].
*   **When to Use:** Suitable for large-scale volume visualization, especially with petavoxel data or highly anisotropic data [S1].
*   **When Not to Use:** Not specified in the sources.

## Background
*   **Origin:** The concept was introduced by Crassin et al. in 2009 with the Gigavoxels system [S2, unofficial]. Gobbetti et al. [GMG08] were among the first to implement a volume ray-caster with stackless GPU octree traversal using occlusion queries, though it was not fully ray-guided because culling was performed on the CPU [S1].
*   **History and Development:**
    *   2009: Crassin et al. [CN09] introduced the Gigavoxels system for GPU-based octree volume rendering with ray-guided streaming [S1].
    *   2012: Hadwiger et al. [HBJP12] proposed a multi-resolution virtual memory scheme based on a multi-level page table hierarchy [S1].
    *   2013: Fogal et al. [FSK13] performed an in-depth analysis of several aspects of ray-guided volume rendering [S1]. Reichl et al. [RTW13] also employed a similar ray-guided approach for large smooth particle hydrodynamics (SPH) simulations [S1].
*   **Where it is used:** Used in large-scale GPU-based volume visualization, including entertainment applications (Gigavoxels) and scientific visualization (electron microscopy, SPH simulations) [S1].

## Sources
[S1] State-of-the-Art in GPU-Based Large-Scale Volume Visualization, https://vcg.seas.harvard.edu/publications/20150513-stateof-the-art-in-gpu-based-large-scale-volume-visualization/paper, fetched 2026-10-24
[S2] GPU-Based Large-Scale Scientific Visualization, https://johanna-b.github.io/LargeSciVis2018/slides/LargeScaleVisualization_Architectures.pdf, fetched 2026-10-29 (unofficial)
