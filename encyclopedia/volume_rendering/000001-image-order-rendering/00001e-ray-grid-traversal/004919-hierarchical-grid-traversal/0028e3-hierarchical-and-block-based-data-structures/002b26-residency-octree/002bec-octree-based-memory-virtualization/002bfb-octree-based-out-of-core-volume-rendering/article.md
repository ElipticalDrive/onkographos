# Octree-based out-of-core volume rendering

Also called: Octree approach, Octree reference implementation, Octree-based Out-of-Core LOD, Out-of-core octree

## Overview
Octree-based out-of-core volume rendering is a method for rendering multi-channel volume datasets that exceed available texture memory by storing octree nodes and bricks on disk and loading them on demand [S1, unofficial][S2, unofficial]. It extends the single-channel approach of Crassin et al. to support multiple visible channels by storing pointers and culling metadata in each node [S1, unofficial]. Unlike the [Residency Octree](5269-residency-octree.md), which uses an octree for hierarchical culling but allows partial subtree residency, this method enforces that the whole subtree is resident in the cache for high-resolution rendering, leading to a higher memory footprint and worse performance for multiple channels compared to the author's page-table method [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Multi-channel volume dataset (up to $n$ channels), camera parameters, transfer function, cache size limit [S1, unofficial].
- Output: Rendered image [S1, unofficial].

**Data Structures:**
1.  **Octree Node:**
    -   `bounding_box`: 3D bounding box of the voxel region [S2, unofficial].
    -   `min_values`: Array of size $m$ (number of visible channels), storing the minimum scalar value for each channel within this node [S1, unofficial].
    -   `max_values`: Array of size $m$, storing the maximum scalar value for each channel within this node [S1, unofficial].
    -   `bricks`: Array of size $m$, where each entry is a pointer (or disk offset) to the brick data for the corresponding channel [S1, unofficial].
    -   `children`: Pointers to 8 child nodes (if not a leaf) [S1, unofficial].
2.  **Hierarchy Structure File:**
    -   Contains connectivity information and visibility culling information (bounding boxes) [S2, unofficial].
    -   Must fit in memory; trivially small [S2, unofficial].
3.  **Brick Cache:**
    -   Fixed-size cache (e.g., 4 GB) holding resident bricks [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:**
    -   Load the hierarchy structure file into memory [S2, unofficial].
    -   Maintain a brick cache with a fixed capacity (e.g., 4 GB) [S1, unofficial].
    -   Start tree traversal at the third subdivision level counting from the root node [S1, unofficial].
    -   Initialize the current traversal node to the parent node’s subdivision level in the tree structure of the last node visited for the last sample [S1, unofficial].
2.  **Ray Traversal and Node Loading:**
    -   For each ray sample, determine the current node in the octree using a full octree implementation that allows direct access to the correct node for a ray sample in each subdivision level [S1, unofficial].
    -   For each visible channel $c \in \{0, \dots, m-1\}$:
        -   Check if the node's brick for channel $c$ is in the cache [S1, unofficial].
        -   If not, load the brick from disk into the cache, evicting old bricks if necessary [S2, unofficial].
        -   Restart the tree traversal for each visible channel to ensure that the whole subtree is resident in the cache for high-resolution rendering [S1, unofficial]. This means traversing from the current node down to the leaf level for channel $c$ to guarantee all required bricks are loaded [S1, unofficial].
        -   If the brick for the desired resolution is missing, render the next lower resolution instead [S1, unofficial].
3.  **Culling:**
    -   Use the `min_values` and `max_values` stored in each node to perform visibility culling [S1, unofficial].
    -   If the scalar range of a node does not intersect with the transfer function's opacity range for any channel, skip the node and its children [S1, unofficial].
    -   Note: Bricks containing at least one non-zero value are not considered EMPTY and must be kept in memory even if transparent under current viewing conditions [S1, unofficial].
4.  **Rendering:**
    -   Sample the loaded brick data for the current ray position [S1, unofficial].
    -   Apply transfer function and compositing [S1, unofficial].
    -   Accumulate color and opacity [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class OctreeNode:
    def __init__(self, level, num_channels):
        self.level = level
        self.children = [None] * 8
        self.bricks = [None] * num_channels
        self.min_values = [0.0] * num_channels
        self.max_values = [0.0] * num_channels
        self.bounding_box = (0.0, 0.0, 0.0, 1.0, 1.0, 1.0)

class Ray:
    def __init__(self, origin, direction):
        self.origin = origin
        self.direction = direction

def get_child_index(node, ray):
    """Determines which of the 8 octree children the ray enters next."""
    # Standard ray-octree child intersection logic
    # Returns index 0-7 based on ray direction and node center
    center = [(node.bounding_box[i] + node.bounding_box[i+3]) / 2.0 for i in range(3)]
    idx = 0
    for i in range(3):
        if ray.direction[i] > 0:
            idx |= (1 << i)
    return idx

def intersects_transfer_function(min_val, max_val, tf_opacity_min, tf_opacity_max):
    """Checks if the scalar range intersects the transfer function's opacity range."""
    return max_val >= tf_opacity_min and min_val <= tf_opacity_max

def sample_brick(ray, brick_data, brick_origin, brick_size):
    """Samples the loaded brick data for the current ray position."""
    # Standard trilinear interpolation or texture lookup
    return (1.0, 1.0, 1.0, 0.0)  # Returns (R, G, B, A)

def accumulate(color, opacity, current_color, current_opacity):
    """Applies front-to-back compositing."""
    new_opacity = current_opacity + opacity * (1.0 - current_opacity)
    new_color = current_color + color * opacity * (1.0 - current_opacity)
    return new_color, new_opacity

def render_octree_out_of_core(hierarchy, cache_size, num_channels, tf_opacity_min, tf_opacity_max, max_level):
    cache = {}  # Simple dict-based cache: key=brick_id, value=brick_data
    start_level = 3
    current_node = hierarchy.root
    current_level = 0
    
    for ray in hierarchy.rays:
        # Ensure we are at least at start_level
        if current_level < start_level:
            node = current_node
            level = current_level
            while level < start_level:
                child_idx = get_child_index(node, ray)
                node = node.children[child_idx]
                level += 1
            current_node = node
            current_level = level
            
        # For each visible channel, ensure full subtree residency
        for c in range(num_channels):
            node = current_node
            level = current_level
            
            # Traverse down to leaf, loading bricks and culling
            while level < max_level:
                # Load brick if missing
                if node.bricks[c] is not None and node.bricks[c] not in cache:
                    if len(cache) >= cache_size:
                        cache.pop(next(iter(cache)))  # Simple eviction
                    cache[node.bricks[c]] = load_brick_from_disk(node.bricks[c])
                
                # Culling check
                if not intersects_transfer_function(node.min_values[c], node.max_values[c], tf_opacity_min, tf_opacity_max):
                    break  # Skip this branch
                
                # Move to child containing ray
                child_idx = get_child_index(node, ray)
                node = node.children[child_idx]
                level += 1
            
            # Sample leaf brick
            if node.bricks[c] in cache:
                brick_data = cache[node.bricks[c]]
                color, opacity = sample_brick(ray, brick_data, node.bounding_box, 32)
                current_color, current_opacity = accumulate(color, opacity, current_color, current_opacity)
            else:
                # Fallback to lower resolution if missing
                lower_brick = get_lower_resolution_brick(node, c)
                color, opacity = sample_brick(ray, lower_brick, node.bounding_box, 32)
                current_color, current_opacity = accumulate(color, opacity, current_color, current_opacity)
                
        # Update current node for next ray optimization
        current_node = node
        current_level = level
```

**Parameters:**
-   Cache size: 4 GB [S1, unofficial].
-   Brick size (and approximate leaf-node size): $32^3$ voxels [S1, unofficial].
-   Hierarchy structure file size: Must fit in memory, trivially small [S2, unofficial].

**Edge Cases and Pitfalls:**
-   **Multiple Channels Performance:** For multiple channels, the octree has to be traversed for each channel to ensure that all subtrees are resident in the cache, leading to worse performance compared to single-channel rendering [S1, unofficial].
-   **Memory Footprint:** Whole subtrees have to be resident in the cache for rendering the volume in high resolution, resulting in a higher memory footprint in terms of the number of bricks in the working set compared to the residency octree [S1, unofficial].
-   **Empty Space Skipping:** Bricks containing at least one non-zero value are not considered EMPTY and thus have to be kept in memory even though they might be completely transparent under the current viewing conditions [S1, unofficial].
-   **Missing Bricks:** In case of a missing brick in the desired resolution, render the next lower resolution instead [S1, unofficial].

## Subtopics

### [Multi-Channel Page-Table Hierarchy](5196-multi-channel-page-table-hierarchy.md)
A GPU-driven data structure that virtualizes a bricked multi-resolution, multi-channel volume via a paged 3D address space. It solves the problem of managing brick residency and performing address translation for out-of-core volume rendering without traversing a tree structure. Unlike the Octree-based out-of-core volume rendering, which uses an octree for hierarchical culling and requires full subtree residency, the page-table hierarchy treats each brick as an independent chunk, allowing direct access to any brick in the dataset. It extends the single-channel page-table approach by maintaining one page-table hierarchy per channel, all sharing a single brick cache.

## Operating characteristics
-   **Time Complexity:** $O(N)$ where $N$ is the number of ray samples, but with a high constant factor due to repeated tree traversals for each channel [S1, unofficial]. For multiple channels, performance degrades significantly compared to single-channel rendering [S1, unofficial].
-   **Memory Complexity:** High. Requires storing the entire subtree for high-resolution rendering in the cache, leading to a larger working set than the residency octree [S1, unofficial]. Cache size is typically 4 GB [S1, unofficial].
-   **Measured Performance:**
    -   CyCIF Small (16 channels): 122.8 ms/frame [S1, unofficial].
    -   CyCIF Medium (15 channels): 155.1 ms/frame [S1, unofficial].
    -   CyCIF Large (4 channels): 123.5 ms/frame [S1, unofficial].
    -   Kingsnake (1 channel): 11.6 ms/frame [S1, unofficial].
    -   Bonsai (1 channel): 4.5 ms/frame [S1, unofficial].
    -   Aneurism (1 channel): 4.8 ms/frame [S1, unofficial].
-   **Scaling:** Performance scales poorly with the number of visible channels due to the need to restart tree traversal for each channel [S1, unofficial].
-   **Strengths:** Simple hierarchical culling using min/max values; effective for single-channel rendering where it achieves comparable frame rates to other methods [S1, unofficial].
-   **Limitations:** Poor performance for multiple channels; high memory footprint due to full subtree residency requirement; inefficient empty-space skipping for non-zero but transparent voxels [S1, unofficial].

## Background
The octree-based out-of-core volume rendering approach is based on the out-of-core renderer presented by Crassin et al. [CNLE09] and is similar to the octree-based method discussed by Brix et al. [BPH14] [S1, unofficial]. It was extended by the author of [S1, unofficial] to support multiple visible channels by storing pointers and culling metadata in each node. The method is part of a broader effort to render large biomedical datasets, such as those generated by Cyclic Immunofluorescence (CyCIF) [LIW+ 18], which can exceed available texture memory [S1, unofficial]. The approach is also related to techniques for visualizing unstructured grids by converting them into a hierarchy of regular grids stored in an octree [S3, unofficial].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S2] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
[S3] PDF A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
