# Sparse Octree Radiance Cache

Also called: Octree GI cache

## Overview
The Sparse Octree Radiance Cache is a CPU-built hierarchical data structure that replaces dense mip-mapped volume grids for storing aggregated radiance, albedo, alpha, and emission data [S1, unofficial]. It solves the resolution loss inherent in uniform grids, where thin features (e.g., 3-voxel thick objects) are averaged into indistinct smears [S1, unofficial]. Unlike its sibling [Sparse Voxel Octree for Indirect Illumination](8483-sparse-voxel-octree-for-indirect-illumination.md), which focuses on GPU-side geometry visibility, this method provides a sparse, DAG-deduplicated representation optimized for cone-integration semantics on the CPU [S1, unofficial].

## How it works
**Inputs:**
- Scene dimensions (e.g., $640 \times 640 \times 640$).
- Voxel data (radiance, albedo, alpha, emission).
- Leaf block size parameter $L$ (e.g., small enough to preserve thin features).

**Outputs:**
- A sparse octree where leaves store aggregated block data and interior nodes are DAG-deduplicated.

**Data Structures:**
- **Leaf Node:** Stores aggregated radiance, albedo, alpha, and emission for a block of $L^3$ voxels.
- **Interior Node:** Contains pointers to up to 8 child octants. Empty octants produce no nodes. A chain of single-child levels is deduplicated to a single 2-word node [S1, unofficial].
- **DAG Structure:** Interior nodes with identical content share the same memory reference [S1, unofficial].

**Steps:**
1. **Enclose Scene:** Pad non-power-of-two scene dimensions to the next power-of-two cube (e.g., 640 $\rightarrow$ 1024). The padded region produces no nodes [S1, unofficial].
2. **Build Tree:** Recursively subdivide the scene. At each level, aggregate voxel data into blocks of size $L$.
3. **Deduplicate:** Apply DAG deduplication to interior nodes. Identical regions (e.g., repeated bookshelf shelves) share nodes [S1, unofficial].
4. **Cone Level Selection:** During traversal, determine the node level based on the cone's cross-section radius, which grows with distance. A far sample stops at a coarse node; a tight cone reaches the leaf. No mip chain or LOD parameter is used [S1, unofficial].

**Formulas:**
- **Leaf Grid Size:** For a scene of width $W$ enclosed in $P$ (power-of-two), and leaf block size $L$, the leaf grid resolution is $P/L$.
- **Interior Levels:** The number of interior levels is $\log_2(P/L)$ [S1, unofficial].

**Parameters:**
- **Leaf Size ($L$):** A parameter, not hard-coded. Must be small enough to preserve thin features (e.g., $L$ such that a 3-voxel leg is not averaged away) [S1, unofficial].

**Pitfalls:**
- **Empty Octants:** Must produce no nodes to maintain sparsity. A chain of 7 single-child levels dedupes to ONE 2-word node [S1, unofficial].
- **Non-Power-of-Two:** Incorrect handling of scene dimensions leads to incorrect node counts or padding artifacts [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_sparse_octree(voxel_data, scene_width, leaf_block_size):
    # Step 1: Enclose scene in power-of-two cube
    enclosing_size = next_power_of_two(scene_width)
    padded_data = pad_to_cube(voxel_data, enclosing_size)
    
    # Step 2: Calculate grid dimensions
    leaf_grid_res = enclosing_size // leaf_block_size
    interior_levels = log2(leaf_grid_res)
    
    # Step 3: Recursive build with DAG deduplication
    root = build_node(padded_data, 0, enclosing_size, leaf_block_size, {})
    return root

def build_node(data, level, current_size, leaf_block_size, seen_nodes):
    # Base case: Leaf node
    if level == log2(leaf_grid_res):
        aggregated = aggregate_block(data, leaf_block_size)
        return LeafNode(aggregated)
    
    # Recursive case: Interior node
    children = []
    for octant in 0..7:
        octant_data = get_octant(data, octant, current_size)
        if is_empty(octant_data):
            continue # Empty octants produce no nodes
        child = build_node(octant_data, level + 1, current_size / 2, leaf_block_size, seen_nodes)
        children.append(child)
    
    # DAG Deduplication
    node_key = hash(children)
    if node_key in seen_nodes:
        return seen_nodes[node_key]
    
    new_node = InteriorNode(children)
    seen_nodes[node_key] = new_node
    return new_node

def select_cone_level(cone_origin, cone_direction, cone_radius, node):
    # Cone level selection based on growing cross-section radius
    # No mip chain or LOD parameter
    distance = dot(cone_direction, node.center - cone_origin)
    effective_radius = cone_radius * (1 + distance)
    
    if effective_radius > node.bounding_box_radius:
        return node # Stop at coarse node
    else:
        return traverse_to_leaf(node, cone_origin, cone_direction, effective_radius)
```

## Subtopics

### [Sparse Voxel Octree for Indirect Illumination](8483-sparse-voxel-octree-for-indirect-illumination.md)
A pre-filtered hierarchical voxel representation of scene geometry stored as a dynamic sparse voxel octree on the GPU, used to efficiently estimate visibility and integrate incoming indirect energy without involving the actual mesh.

## Operating characteristics
- **Time Complexity:** Building time depends on the number of non-empty voxels and the depth of the tree ($O(N \log N)$ where $N$ is the number of voxels). Traversal time depends on the cone footprint and tree depth.
- **Memory Complexity:** Significantly reduced compared to dense grids due to sparsity and DAG deduplication. Node count is far below voxel count [S1, unofficial].
- **Strengths:** Preserves thin features; handles non-power-of-two scenes; efficient memory usage via DAG deduplication.
- **Limitations:** CPU-only build (in this implementation); no GPU-side cone rule integration yet [S1, unofficial].
- **When to Use:** When thin features must be preserved and memory efficiency is critical.
- **When Not to Use:** When GPU-side geometry visibility is the primary concern (use [Sparse Voxel Octree for Indirect Illumination](8483-sparse-voxel-octree-for-indirect-illumination.md) instead).

## Background
The Sparse Octree Radiance Cache was proposed by ZhouXiaolin in October 2026 as an enhancement to replace dense uniform grids in GI caching [S1, unofficial]. It addresses the resolution problem where thin features are averaged away in dense grids. The implementation was completed in commit `27e45be` and verified with CPU-side tests [S1, unofficial].

## Sources
[S1] Octree GI cache: CPU builder and cone-integration semantics, https://github.com/ZhouXiaolin/evoxl/issues/1, fetched 2026-10-26 (unofficial)
