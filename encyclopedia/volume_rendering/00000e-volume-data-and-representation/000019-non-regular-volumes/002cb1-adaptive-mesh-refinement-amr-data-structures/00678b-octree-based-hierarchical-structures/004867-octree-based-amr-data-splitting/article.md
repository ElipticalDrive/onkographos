# Octree-based AMR Data Splitting

Also called: Chopping up AMR data set, Overlaying with octree

## Overview
This method partitions hierarchical Adaptive Mesh Refinement (AMR) data into uniform, rectilinear blocks suitable for parallel rendering by overlaying the domain with an octree aligned to the finest grid resolution [S1, unofficial]. It sets itself apart from sibling methods by generating a static, view-independent octree structure that maps directly to uniform blocks, enabling straightforward hardware-accelerated texture uploads [S1, unofficial].

## How it works
**Inputs:** An AMR dataset containing a hierarchy of grid levels with varying resolutions [S1, unofficial].
**Outputs:** A set of uniform rectilinear grid blocks, each corresponding to an octree leaf, ready for distribution to processing nodes [S1, unofficial].

**Data Structures:**
*   **Octree Node:** Contains a bounding box aligned with the finest grid, a boolean flag indicating if it is a leaf, and a list of child nodes (if not a leaf) [G].
*   **Uniform Block:** A 3D array of scalar values corresponding to the data within the octree leaf's bounding box, aligned to the finest grid resolution [S1, unofficial].

**Steps:**
1.  **Overlay:** Define an octree structure over the entire data domain, aligned such that its finest possible resolution matches the finest grid cells in the AMR hierarchy [S1, unofficial].
2.  **Recursive Splitting:** For each octree node, determine if it is completely overlaid by a single AMR grid level. If the node's bounding box is fully contained within a single grid level and all data values within that box are defined by that grid, it becomes a leaf node. Otherwise, split it into 8 children by subdividing the bounding box equally in all three dimensions and recurse [S1, unofficial].
3.  **Leaf Extraction:** For every leaf node in the resulting octree, generate a uniform 3D volume block containing the scalar values from the underlying AMR grid that fall within the leaf's bounds [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class OctreeNode:
    def __init__(self, bounds, depth=0):
        self.bounds = bounds
        self.depth = depth
        self.is_leaf = False
        self.children = []
        self.data = None

def split_amr_to_octree(amr_data, finest_resolution):
    root = OctreeNode(bounds=amr_data.domain_bounds, depth=0)
    build_octree(root, amr_data, finest_resolution)
    uniform_blocks = []
    collect_leaves(root, uniform_blocks)
    return uniform_blocks

def build_octree(node, amr_data, finest_resolution):
    overlaid_grid = check_single_grid_overlay(node.bounds, amr_data)
    if overlaid_grid is not None:
        node.is_leaf = True
        node.data = extract_uniform_data(overlaid_grid, node.bounds, finest_resolution)
    else:
        node.is_leaf = False
        child_bounds = split_bounds(node.bounds)
        for cb in child_bounds:
            child = OctreeNode(bounds=cb, depth=node.depth + 1)
            node.children.append(child)
            build_octree(child, amr_data, finest_resolution)

def check_single_grid_overlay(bounds, amr_data):
    for level in amr_data.levels:
        if bounds_contained_in(level.bounds, bounds) and cells_intersect(level, bounds):
            return level
    return None

def extract_uniform_data(grid_level, bounds, finest_resolution):
    dims = calculate_dims(bounds, finest_resolution)
    data_array = [[[0] * dims[2] for _ in range(dims[1])] for _ in range(dims[0])]
    for i, j, k in grid_level.cells_within(bounds):
        u, v, w = map_to_uniform(i, j, k, finest_resolution)
        if 0 <= u < dims[0] and 0 <= v < dims[1] and 0 <= w < dims[2]:
            data_array[u][v][w] = grid_level.values[i, j, k]
    return data_array

def collect_leaves(node, blocks):
    if node.is_leaf:
        blocks.append(node.data)
    else:
        for child in node.children:
            collect_leaves(child, blocks)

def split_bounds(bounds):
    min_x, min_y, min_z, max_x, max_y, max_z = bounds
    mid_x, mid_y, mid_z = (min_x + max_x) / 2, (min_y + max_y) / 2, (min_z + max_z) / 2
    return [
        (min_x, min_y, min_z, mid_x, mid_y, mid_z),
        (mid_x, min_y, min_z, max_x, mid_y, mid_z),
        (min_x, mid_y, min_z, mid_x, max_y, mid_z),
        (mid_x, mid_y, min_z, max_x, max_y, mid_z),
        (min_x, min_y, mid_z, mid_x, mid_y, max_z),
        (mid_x, min_y, mid_z, max_x, mid_y, max_z),
        (min_x, mid_y, mid_z, mid_x, max_y, max_z),
        (mid_x, mid_y, mid_z, max_x, max_y, max_z)
    ]

def bounds_contained_in(grid_bounds, node_bounds):
    return (grid_bounds[0] <= node_bounds[0] and grid_bounds[3] >= node_bounds[3] and
            grid_bounds[1] <= node_bounds[1] and grid_bounds[4] >= node_bounds[4] and
            grid_bounds[2] <= node_bounds[2] and grid_bounds[5] >= node_bounds[5])

def cells_intersect(grid_level, bounds):
    return (grid_level.bounds[0] <= bounds[3] and grid_level.bounds[3] >= bounds[0] and
            grid_level.bounds[1] <= bounds[4] and grid_level.bounds[4] >= bounds[1] and
            grid_level.bounds[2] <= bounds[5] and grid_level.bounds[5] >= bounds[2])

def calculate_dims(bounds, finest_resolution):
    min_x, min_y, min_z, max_x, max_y, max_z = bounds
    return (int((max_x - min_x) * finest_resolution),
            int((max_y - min_y) * finest_resolution),
            int((max_z - min_z) * finest_resolution))

def map_to_uniform(i, j, k, finest_resolution):
    # Standard coordinate-to-index mapping based on grid origin and resolution [G]
    return (i, j, k)
```

**Parameters:**
*   **Finest Resolution:** The maximum grid resolution in the AMR hierarchy. Typical value: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Partial Overlays:** If a node's bounds partially overlap with a grid cell, the recursive splitting continues until a fully contained single-grid region is found [S1, unofficial].
*   **Precision:** Aligning the octree with the finest grid cells requires precise floating-point arithmetic to avoid gaps or overlaps between blocks [G].

## Subtopics

### Back-to-front Cell Assignment
A parallel distribution strategy that breaks octree leaves down to the cell level, numbers them in viewing direction, and assigns contiguous ranges of cells to processing nodes for good load balancing [S1, unofficial]. It sets itself apart from [Round-robin Leaf Assignment](12287-round-robin-leaf-assignment.md) by providing better load balancing and simpler compositing, at the cost of view-dependent I/O overhead [S1, unofficial].

**Steps:**
1.  Break octree leaves down to the cell level.
2.  Traverse the resulting octree in viewing direction and number leaves on-the-fly.
3.  In a second traversal, assign each processing node $i$ a range of leaves to render.

**Formula:**
Node $i$ renders leaves from index $(N*i)/n$ to $(N*(i+1))/n$, where $N$ is the total number of octree leaves and $n$ is the number of processing nodes [S1, unofficial].

**Pitfalls:**
Assignment depends on viewing direction, which might increase I/O overhead during viewpoint animation [S1, unofficial].

**Comparison:**
This approach provides very good load balancing because rendering cost can be estimated accurately, unlike round-robin leaf assignment [S1, unofficial]. Compositing is extremely simple because partial images can be composited in order of processing node numbers, unlike round-robin [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The method is described as "very simple and efficient" [S1, unofficial]. Hardware acceleration is used for rendering uniform blocks by uploading them as stacks of axis-aligned 2D textures [S1, unofficial].
*   **Strengths:** Converts unstructured AMR data into uniform blocks, which are easy to handle in parallel rendering pipelines [S1, unofficial].
*   **Limitations:** The uniform blocks may not perfectly match the underlying AMR structure, potentially leading to some data duplication [S1, unofficial].
*   **When to use:** When rendering AMR data on parallel hardware that benefits from uniform data blocks, such as GPUs using texture-based rendering [S1, unofficial].
*   **When not to use:** Not given in the sources.

## Background
The method was developed in the context of volume visualization of AMR data, specifically for parallel rendering on graphics workstations. It is part of the AMRVis project, which investigates efficient methods for visualizing adaptive mesh refinement data [S1, unofficial]. The approach leverages the simplicity of uniform grids to facilitate hardware-accelerated rendering while maintaining the adaptive resolution benefits of AMR [S1, unofficial].

## Sources
[S1] Volume Visualization of AMR Data, https://graphics.cs.ucdavis.edu/~okreylos/Research/AMRVis/MainPage.html, fetched 2026-10-26 (unofficial)
