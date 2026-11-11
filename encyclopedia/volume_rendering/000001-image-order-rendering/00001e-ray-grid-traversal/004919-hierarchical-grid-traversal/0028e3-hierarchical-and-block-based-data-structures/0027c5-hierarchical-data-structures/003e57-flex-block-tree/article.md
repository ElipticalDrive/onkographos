# Flex-block tree

Also called: Cell-tree, Flex-block partition

## Overview
The flex-block tree is a hierarchical spatial subdivision structure derived from a kd-tree, designed primarily for load balancing in distributed volume rendering (both ray casting and ray tracing) [S1, unofficial]. Unlike traditional hierarchical structures that enforce strict axis-alternating splits, the flex-block partition allows flexible cut orientations to accommodate non-empty subvolume boundaries and render-node constraints [S1, unofficial]. It serves as a dependency graph that determines the deterministic depth order of cells for image compositing in ray casting and accelerates ray traversal by indicating potential neighbor cells in ray tracing [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A massive volume dataset, optionally with a pre-computed mask volume defining regions of interest [S1, unofficial].
*   **Output:** A flex-block tree (dependency graph) and associated partition data distributed across render nodes [S1, unofficial].

**Data Structures:**
1.  **Flex-block Tree Node:** A binary tree node containing cut plane information (position and orientation) and child node pointers [S1, unofficial]. The structure does not restrict partition cuts to alternate between axes [S1, unofficial].
2.  **Flex-block (Leaf Node):** Represents a cell in the partition [S1, unofficial]. It contains a tightly cropped subvolume that does not necessarily occupy the entire geometric cell defined by the tree node [S1, unofficial]. It is defined by an Axis-Aligned Bounding Box (AABB) that tightly encloses the non-empty data within the cell [S1, unofficial].

**Construction Steps:**
1.  **Preprocessing (Data Reduction):** One raw data slice is read at a time and out-of-core region growing creates a mask for each slice, or a provided mask volume is used [S1, unofficial]. Slices are cropped to one or more bounding rectangles of regions of interest using the mask volume [S1, unofficial].
2.  **Partitioning (Moving Walls):** A moving walls algorithm partitions the scene [S1, unofficial]. Partition walls are moved to simultaneously accommodate non-empty subvolume boundaries and render-node assignments for load balancing [S1, unofficial]. Dynamic programming solves the Load Balanced Network Distribution (LBND) optimization problem to minimize end-to-end rendering time [S1, unofficial].
3.  **Tree Extraction:** A kd-tree is extracted from the output of the moving walls procedure [S1, unofficial]. This kd-tree forms the basis of the flex-block tree [S1, unofficial].
4.  **Leaf Definition:** Leaf nodes of the tree are defined as flex-blocks [S1, unofficial]. Each flex-block contains a tightly cropped subvolume aligned with a tight AABB [S1, unofficial].
5.  **Dependency Graph Generation:** The image priority order (depth order) for eight quantized view directions is extracted from the kd-tree [S1, unofficial]. This order is stored in a look-up table that represents the dependency graph of flex-blocks [S1, unofficial].
6.  **Distribution:** Flex-block parameters (including the dependency graph) and cropped slices (or pre-created volume blocks) are distributed across the cluster along with render-node assignments [S1, unofficial].

**Ray Traversal and Scheduling (Ray Tracing Context):**
1.  **Initialization:** Data is distributed using the flex-block partition [S1, unofficial].
2.  **Scheduling:** Ray-task scheduling is updated dynamically within the local partition of each render node [S1, unofficial].
3.  **Traversal Loop:** A ray queue holds pending rays for each cell [S1, unofficial]. One cell is worked on at a time [S1, unofficial]. When a cell is in main memory, each ray in the corresponding ray queue is traced through that cell [S1, unofficial]. Space skipping within each cell is automated by advancing the ray forward according to the data extents (AABB) of the current flex-block [S1, unofficial]. When a spawned ray exits the current cell, it is placed on the queue of the next cell it intersects [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_flex_block_tree(volume_data, mask_volume=None):
    # 1. Preprocessing: out-of-core region growing for mask
    if mask_volume is None:
        mask_volume = out_of_core_region_growing(volume_data)
    # Crop slices to bounding rectangles of regions of interest
    cropped_slices = crop_slices_to_bounding_rects(volume_data, mask_volume)

    # 2. Partitioning: moving walls algorithm with dynamic programming for LBND
    partition_result = moving_walls_partition(cropped_slices)

    # 3. Tree Extraction: extract kd-tree from moving walls output
    kd_tree = extract_kd_tree(partition_result)

    # 4. Leaf Definition: leaf nodes are flex-blocks
    flex_block_tree = FlexBlockTree(kd_tree)

    # 5. Dependency Graph: image priority order for 8 quantized view directions
    view_directions = get_quantized_view_directions(8)
    dependency_graph = {}
    for view_dir in view_directions:
        order = extract_image_priority_order(flex_block_tree, view_dir)
        dependency_graph[view_dir] = order

    return flex_block_tree, dependency_graph

def trace_rays_ray_tracing(flex_block_tree, dependency_graph, rays, render_node_id):
    # 1. Distribute data
    local_data = distribute_data(flex_block_tree, render_node_id)

    # 2. Initialize ray queues for each cell
    ray_queues = {cell: [] for cell in flex_block_tree.cells}
    for ray in rays:
        entry_cell = flex_block_tree.get_cell_containing(ray.origin)
        ray_queues[entry_cell].append(ray)

    # 3. Dynamic scheduling & traversal
    while any(ray_queues.values()):
        current_cell = select_next_cell(ray_queues, dependency_graph)
        if current_cell in local_data:
            for ray in ray_queues[current_cell]:
                # Space skipping: advance ray according to flex-block extents
                cell_aabb = flex_block_tree.get_cell_aabb(current_cell)
                ray.origin = max(ray.origin, cell_aabb.min)

                hit, new_ray = trace_ray_through_cell(ray, current_cell)
                if hit:
                    next_cell = flex_block_tree.get_next_cell(ray, current_cell)
                    if next_cell:
                        ray_queues[next_cell].append(new_ray)

# Helper definitions (standard operations)
def out_of_core_region_growing(volume_data):
    return generate_mask_from_volume(volume_data)

def crop_slices_to_bounding_rects(volume_data, mask_volume):
    return [slice.crop(mask) for slice in volume_data]

def moving_walls_partition(cropped_slices):
    return apply_moving_walls_and_dp(cropped_slices)

def extract_kd_tree(partition_result):
    return build_kd_tree_from_partition(partition_result)

class FlexBlockTree:
    def __init__(self, kd_tree):
        self.kd_tree = kd_tree
        self.cells = [node for node in kd_tree.nodes if node.is_leaf]
    def get_cell_containing(self, point):
        return self.kd_tree.find_leaf(point)
    def get_cell_aabb(self, cell):
        return cell.aabb
    def get_next_cell(self, ray, current_cell):
        return self.kd_tree.find_next_intersected_cell(ray, current_cell)

def get_quantized_view_directions(n):
    return standard_quantized_directions(n)

def extract_image_priority_order(tree, view_dir):
    return tree.get_depth_order(view_dir)

def distribute_data(tree, node_id):
    return tree.assign_to_node(node_id)

def select_next_cell(queues, graph):
    return next(iter(queues))

def trace_ray_through_cell(ray, cell):
    return ray.intersect(cell)
```

**Parameters:**
*   **View Directions:** Eight quantized view directions are used to extract the image priority order [S1, unofficial].
*   **Load Balancing:** The partitioning aims to minimize end-to-end rendering time, which is a function of the slowest render-node, image composition time, and network communication time [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **NP-Completeness:** Determining the optimum order for caching cells in ray tracing is NP-Complete [S1, unofficial].
*   **Single Fetch Limitation:** It is not generally possible to create a schedule which fetches each cell only once for ray tracing algorithms [S1, unofficial].
*   **Disproportionate Scenes:** The lack of axis-alternating restriction is critical for scenes with disproportionate lengths on each side [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Construction involves region growing, cropping, and kd-tree extraction. The complexity of optimal caching is NP-Complete [S1, unofficial].
*   **Memory Complexity:** The cell-tree (dependency graph) is described as being several orders of magnitude smaller than the number of dependencies it represents [S1, unofficial].
*   **Strengths:** Specifically designed for distributed rendering to balance load across render nodes [S1, unofficial]. Early cropping of empty space reduces run-time for preprocessing, processing, and network movement [S1, unofficial]. Provides a deterministic depth order for image compositing in ray casting [S1, unofficial]. Accelerates ray tracing by indicating potential neighbors and automating space skipping [S1, unofficial].
*   **Limitations:** Cannot guarantee single-fetch scheduling for all cells [S1, unofficial]. Requires complex preprocessing (moving walls, dynamic programming for load balancing) [S1, unofficial].

## Background
The flex-block tree and partition were introduced in the context of massive data management for distributed volume visualization [S1, unofficial]. The work addresses the Load Balanced Network Distribution (LBND) problem, an optimization problem for minimizing end-to-end rendering time in distributed clusters [S1, unofficial]. It builds upon traditional space subdivision methods (like kd-trees) but modifies them to address load balancing and empty space reduction specifically for distributed environments [S1, unofficial]. The approach supports both software ray tracing and hardware ray casting [S1, unofficial].

## Sources
[S1] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
