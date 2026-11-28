# Flex-block tree

Also called: Cell-tree, Flex-block partition

## Overview
The flex-block tree is a hierarchical data structure derived from a kd-tree partition, designed primarily for load balancing in distributed volume rendering (both ray casting and ray tracing) [S1, unofficial]. It partitions a volume into "flex-blocks," which are cells containing tightly cropped subvolumes that do not necessarily occupy the entire cell volume [S1, unofficial]. Unlike traditional space subdivision methods, the flex-block partition does not restrict partition cuts to alternate between axes, allowing it to handle scenes with disproportionate dimensions effectively [S1, unofficial]. It serves as a dependency graph that determines the deterministic depth order of blocks for image compositing in ray casting and accelerates ray traversal by indicating potential neighbor cells in ray tracing [S1, unofficial].

## How it works
The method constructs a tree where internal nodes define partition boundaries and leaf nodes represent the data containers (flex-blocks) [S1, unofficial]. The construction involves preprocessing the volume to remove empty space, partitioning the remaining data using a modified kd-tree algorithm, and extracting view-dependent ordering information [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   Raw volume data (slices or voxels) [S1, unofficial].
*   Optional mask volume defining regions of interest [S1, unofficial].
*   Number of render-nodes (for distributed systems) [S1, unofficial].
*   View directions (typically 8 quantized directions) [S1, unofficial].

**Outputs:**
*   A flex-block tree (dependency graph) [S1, unofficial].
*   Cropped volume data (bricks) distributed to render-nodes [S1, unofficial].
*   Look-up tables mapping view directions to flex-block traversal orders [S1, unofficial].

### Data Structures

**Flex-block Tree Node**
Internal nodes in the tree store partitioning information similar to a traditional binary tree [S1, unofficial]:
*   `cut_plane`: The position and normal of the splitting plane.
*   `children`: Pointers or indices to the left and right child nodes.

**Flex-block (Leaf Node)**
Each leaf node represents a cell in the partition [S1, unofficial]:
*   `bbox`: A tightly cropped axis-aligned bounding box (AABB) enclosing the non-empty data within the cell.
*   `data_ref`: Reference to the subvolume data (brick) stored in memory or on disk.
*   `dependencies`: Information used for ray-task scheduling and traversal speedup [S1, unofficial].

### Construction Steps

1.  **Empty Space Reduction:**
    *   If a mask volume is not provided, perform out-of-core region growing on raw data slices to create a mask [S1, unofficial].
    *   Crop each slice to bounding rectangles of non-empty regions defined by the mask [S1, unofficial].
    *   Align a tight AABB around cohesive non-empty areas of the volume to eliminate empty space [S1, unofficial].

2.  **Partitioning (Moving Walls):**
    *   Apply a "moving walls" algorithm to partition the scene [S1, unofficial].
    *   This algorithm moves partition walls to simultaneously accommodate non-empty subvolume boundaries and balance the load across render-nodes [S1, unofficial].
    *   This step generates a partition structure that respects the data distribution requirements [S1, unofficial].

3.  **kd-tree Extraction:**
    *   Extract a kd-tree from the output of the moving walls procedure [S1, unofficial].
    *   Unlike standard kd-trees, this partition does not enforce alternating axis cuts [S1, unofficial].
    *   The cut axis is chosen to optimize load balancing and accommodate disproportionate scene dimensions [S1, unofficial].

4.  **Dynamic Programming for Load Balancing:**
    *   Use a dynamic programming (DP) algorithm to group bricks and solve the Load Balanced Network Distribution (LBND) problem [S1, unofficial].
    *   The DP evaluates a cost function to minimize end-to-end rendering time, which is a function of the slowest render-node, image composition time, and network communication time [S1, unofficial].
    *   This step creates the final flex-blocks within the kd-tree-like partition [S1, unofficial].

5.  **Dependency Graph Construction:**
    *   For each of the eight quantized view directions, extract the image priority order (depth order) of the flex-blocks from the kd-tree [S1, unofficial].
    *   Store these orders in look-up tables [S1, unofficial].
    *   These tables form the dependency graph of flex-blocks [S1, unofficial].
    *   This graph is used for ray casting to determine the order for compositing images from different nodes [S1, unofficial].
    *   This graph is used for ray tracing to accelerate traversal by indicating potential neighbors a ray may enter [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_flex_block_tree(volume_data, mask=None, num_nodes=1, view_directions=8):
    """
    Constructs a flex-block tree for distributed volume rendering.
    
    Args:
        volume_data: Raw volume slices or voxels.
        mask: Optional mask volume. If None, generated via region growing.
        num_nodes: Number of render-nodes for load balancing.
        view_directions: Number of quantized view directions (typically 8).
        
    Returns:
        flex_block_tree: The constructed tree structure.
        dependency_tables: Look-up tables for view-dependent traversal orders.
        flex_blocks: The final partitioned flex-blocks.
    """
    # Step 1: Empty Space Reduction
    if mask is None:
        mask = region_growing(volume_data)
    cropped_slices = crop_slices(volume_data, mask)
    volume_data_reduced = align_tight_bboxes(cropped_slices)
    
    # Step 2: Partitioning (Moving Walls)
    partition_structure = moving_walls_algorithm(volume_data_reduced, num_nodes)
    
    # Step 3: kd-tree Extraction
    kd_tree = extract_kd_tree(partition_structure)
    
    # Step 4: Dynamic Programming for Load Balancing
    flex_blocks = dynamic_programming_partition(kd_tree, num_nodes)
    
    # Step 5: Dependency Graph Construction
    dependency_tables = {}
    for direction in get_quantized_view_directions(view_directions):
        order = get_image_priority_order(kd_tree, flex_blocks, direction)
        dependency_tables[direction] = order
        
    flex_block_tree = build_tree_structure(kd_tree, flex_blocks)
    return flex_block_tree, dependency_tables, flex_blocks

def region_growing(volume_data):
    # Out-of-core region growing to create a mask for each slice [S1, unofficial]
    mask = []
    for slice_data in volume_data:
        mask.append(compute_non_empty_regions(slice_data))
    return mask

def crop_slices(volume_data, mask):
    # Crop each slice to bounding rectangles of non-empty regions [S1, unofficial]
    cropped = []
    for slice_data, m in zip(volume_data, mask):
        cropped.append(slice_data.crop_to_rects(m))
    return cropped

def align_tight_bboxes(cropped_slices):
    # Align a tight AABB around cohesive non-empty areas [S1, unofficial]
    reduced = []
    for slice_data in cropped_slices:
        reduced.append(slice_data.align_bbox())
    return reduced

def moving_walls_algorithm(volume_data_reduced, num_nodes):
    # Move partition walls to accommodate non-empty boundaries and balance load [S1, unofficial]
    return partition_scene(volume_data_reduced, num_nodes)

def extract_kd_tree(partition_structure):
    # Extract kd-tree; does not restrict cuts to alternate axes [S1, unofficial]
    return build_kd_tree(partition_structure, alternate_axes=False)

def dynamic_programming_partition(kd_tree, num_nodes):
    # DP solves LBND problem to minimize end-to-end render time [S1, unofficial]
    return solve_lbnd_dp(kd_tree, num_nodes)

def get_quantized_view_directions(count):
    # Extract 8 quantized view directions [S1, unofficial]
    return generate_quantized_directions(count)

def get_image_priority_order(kd_tree, flex_blocks, direction):
    # Extract depth order for compositing/traversal [S1, unofficial]
    return compute_depth_order(kd_tree, flex_blocks, direction)

def build_tree_structure(kd_tree, flex_blocks):
    # Internal nodes: cut_plane, children. Leaf nodes: flex-block [S1, unofficial]
    return assemble_tree(kd_tree, flex_blocks)
```

### Parameters
*   **View Directions:** Typically 8 quantized directions are used to extract image priority orders [S1, unofficial].
*   **Render-nodes:** The number of processing units in the distributed system, used to balance the partition [S1, unofficial].

### Edge Cases and Pitfalls
*   **Disproportionate Scenes:** Standard kd-trees may perform poorly on scenes with disproportionate dimensions [S1, unofficial]. The flex-block partition addresses this by not restricting cut axes [S1, unofficial].
*   **NP-Completeness:** Determining the optimum order for caching cells in ray tracing is NP-Complete [S1, unofficial]. The method uses dynamic programming approximations [S1, unofficial].
*   **Data Movement:** Reducing empty space early in the pipeline reduces run-time for preprocessing and network data movement [S1, unofficial]. Failure to crop empty space effectively can lead to inefficient load balancing [S1, unofficial].

## Subtopics

### Cell-tree
A compact representation of ray dependencies between cells, where a cell is a flex-block [S1, unofficial]. It is used for ray-task scheduling and ray traversal speedup in ray tracing [S1, unofficial].

### Flex-block Partition
The underlying data distribution method derived from the flex-block tree [S1, unofficial]. It defines how data is split among render-nodes and how bricks are scheduled [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The construction involves region growing, partitioning, and dynamic programming [S1, unofficial]. The DP solution for LBND is optimized to minimize end-to-end rendering time, which depends on the slowest node, composition time, and network communication [S1, unofficial]. The exact complexity of the DP step is not specified, but the problem of optimal caching order is NP-Complete [S1, unofficial].
*   **Memory Complexity:** The cell-tree (dependency graph) is described as being several orders of magnitude smaller than the number of dependencies it represents [S1, unofficial].
*   **Strengths:**
    *   Effective load balancing for distributed rendering [S1, unofficial].
    *   Handles scenes with disproportionate dimensions [S1, unofficial].
    *   Automates space skipping by using data extents of flex-blocks [S1, unofficial].
    *   Reduces data movement by cropping empty space early [S1, unofficial].
*   **Limitations:**
    *   Requires preprocessing (region growing, cropping, partitioning) [S1, unofficial].
    *   Dynamic task scheduling is complex due to the NP-Complete nature of optimal caching [S1, unofficial].
*   **When to Use:** Suitable for distributed volume visualization of massive datasets where load balancing and efficient data distribution are critical [S1, unofficial].

## Background
The flex-block tree was introduced in the context of massive data management for distributed volume visualization [S1, unofficial]. It addresses the limitations of traditional space subdivision methods, which do not adequately handle load balancing in distributed rendering environments [S1, unofficial]. The method was developed to support both software ray tracing and hardware ray casting in a distributed cluster setting [S1, unofficial]. It builds upon concepts from kd-trees and dynamic programming for load balanced network distribution (LBND) [S1, unofficial].

## Sources
[S1] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
