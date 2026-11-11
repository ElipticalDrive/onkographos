# LBVH Construction for Sparse Volumetric Data
Also called: GPU LBVH for Sparse Volumes, Zellmann et al. LBVH

## Overview
This method constructs a Linear Bounding Volume Hierarchy (LBVH) acceleration structure specifically for sparse volumetric data to enable efficient empty-space skipping during ray marching. It adapts the LBVH construction algorithm originally designed for triangle geometry [S1, unofficial]. By using the boundaries of non-empty bricks as bounding geometry, it allows for construction on the GPU at interactive frame rates for large volumes (e.g., $1024^3$) and achieves two to three times higher frame rates compared to ray marching without empty-space skipping [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sparse volumetric dataset represented as a uniform grid of bricks. The grid has a fixed brick size. Non-empty bricks are identified.
*   **Output:** An LBVH data structure where nodes contain bounding volumes enclosing the non-empty bricks in their subtree.

**Data Structures:**
*   **Brick:** A unit of the uniform grid. Each brick is either empty or contains voxel data.
*   **Bounding Volume:** A bounding box enclosing a set of bricks. In the context of adapting triangle LBVH, the "geometry" consists of the boundaries of the non-empty bricks.

**Steps:**
1.  **Grid Partitioning:** Divide the volume into a uniform grid with a fixed brick size [S1, unofficial].
2.  **Identify Non-Empty Bricks:** Determine which bricks contain data.
3.  **Bounding Geometry Generation:** For each non-empty brick, generate its boundary geometry (bounding box) to serve as the primitive set for the LBVH construction [S1, unofficial].
4.  **LBVH Construction:** Apply the linear bounding volume hierarchy construction algorithm (typically a middle-split heuristic on Morton-ordered primitives) to the set of brick boundaries [S1, unofficial]. This step is performed on the GPU to achieve interactive frame rates.

**Parameters:**
*   **Brick Size:** Fixed size for the uniform grid partitioning. Not given in the sources.
*   **Volume Resolution:** Demonstrated for $1024^3$ volumes [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Volumes:** If all bricks are empty, the LBVH construction may result in an empty or trivial structure.
*   **Precision:** GPU-based construction requires careful handling of floating-point precision for bounding box calculations to avoid gaps or overlaps. [G]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_lbvh_sparse_volume(bricks, brick_size):
    """
    Constructs an LBVH for sparse volumetric data.
    
    Args:
        bricks: List of non-empty brick identifiers or their spatial positions.
        brick_size: Fixed size of each brick in the uniform grid.
        
    Returns:
        lbvh_tree: The constructed Linear Bounding Volume Hierarchy.
    """
    # Step 1: Generate bounding geometry for non-empty bricks
    # Each brick is represented by its axis-aligned bounding box (AABB)
    primitives = []
    for brick in bricks:
        # Calculate AABB for the brick based on its grid coordinates and brick_size
        # [G] Assuming brick coordinates are integers (x, y, z)
        min_corner = [brick.x * brick_size, brick.y * brick_size, brick.z * brick_size]
        max_corner = [(brick.x + 1) * brick_size, (brick.y + 1) * brick_size, (brick.z + 1) * brick_size]
        primitives.append(AABB(min_corner, max_corner))
    
    # Step 2: Construct LBVH from primitives
    # [G] This typically involves sorting primitives by Morton code,
    # then recursively splitting the list in the middle to form the hierarchy.
    # The implementation details of the GPU LBVH construction are not provided in the source.
    lbvh_tree = build_lbvh(primitives)
    
    return lbvh_tree
```

## Subtopics

### [Linear Bounding Volume Hierarchy (LBVH) for Volume Rendering](11217-linear-bounding-volume-hierarchy-lbvh-for-volume-rendering.md)
Linear Bounding Volume Hierarchy (LBVH) is a hierarchical data structure for sparse volume rendering that enables empty space skipping, adapted from surface ray tracing to uniform grids and targeted for GPU implementation via CUDA. It sets itself apart from SAH-based methods (such as k-d trees) by using a middle-split heuristic on Morton-ordered bricks, offering significantly superior construction performance at the cost of inferior culling properties.

## Operating characteristics
*   **Performance:** Allows constructing an acceleration structure on the GPU at interactive frame rates for $1024^3$ volumes [S1, unofficial].
*   **Rendering Speed:** The LBVH structure allows for two to three times higher frame rates than using no empty-space skipping during ray marching [S1, unofficial].
*   **Scalability:** Designed for GPU implementation, suitable for large volumes.
*   **Limitations:** As an LBVH, it may have inferior culling properties compared to SAH-based methods like k-d trees [S1, unofficial].

## Background
Zellmann et al. [ZHL19] proposed this adaptation of a linear bounding volume hierarchy (LBVH) construction algorithm for triangle geometry to sparse volumetric data. They used the boundaries of non-empty bricks as bounding geometry. This work is part of a broader effort to support run-time re-construction of acceleration data structures for empty-space skipping in volume rendering [S1, unofficial].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
