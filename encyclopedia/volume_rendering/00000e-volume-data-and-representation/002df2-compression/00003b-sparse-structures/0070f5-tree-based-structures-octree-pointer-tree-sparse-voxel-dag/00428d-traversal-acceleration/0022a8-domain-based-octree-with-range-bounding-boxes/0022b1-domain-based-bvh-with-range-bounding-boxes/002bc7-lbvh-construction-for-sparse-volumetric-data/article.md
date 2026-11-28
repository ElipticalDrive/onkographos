# LBVH Construction for Sparse Volumetric Data
Also called: GPU LBVH for Sparse Volumes, Zellmann et al. LBVH

## Overview
This method constructs a Linear Bounding Volume Hierarchy (LBVH) acceleration structure for sparse volumetric data to enable empty-space skipping during ray marching [S1, unofficial]. It adapts an LBVH construction algorithm originally designed for triangle geometry to sparse volumes by using the boundaries of non-empty bricks as bounding geometry [S1, unofficial]. The approach allows GPU construction at interactive frame rates for $1024^3$ volumes and yields two to three times higher frame rates than ray marching without empty-space skipping [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A sparse volumetric dataset partitioned into a uniform grid with a fixed brick size [S1, unofficial].
*   **Output:** An LBVH data structure where nodes contain bounding volumes enclosing the non-empty bricks in their subtree [G].

**Data Structures:**
*   **Brick:** A unit of the uniform grid. Each brick is either empty or contains voxel data [G].
*   **Bounding Volume:** An axis-aligned bounding box enclosing a set of bricks, derived from the boundaries of the non-empty bricks [S1, unofficial].

**Numbered Steps:**
1.  **Grid Partitioning:** Divide the volume into a uniform grid with a fixed brick size [S1, unofficial].
2.  **Identify Non-Empty Bricks:** Determine which bricks contain data [G].
3.  **Bounding Geometry Generation:** For each non-empty brick, generate its boundary geometry (bounding box) to serve as the primitive set for the LBVH construction [S1, unofficial].
4.  **Hierarchy Construction:** Apply the LBVH construction algorithm to the set of brick boundaries [S1, unofficial]. This step is performed on the GPU to achieve interactive frame rates [S1, unofficial].

**Formulas:**
Not given in the sources.

**Parameters:**
*   **Brick Size:** Fixed size for the uniform grid partitioning. Not given in the sources.
*   **Volume Resolution:** Demonstrated for $1024^3$ volumes [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Volumes:** If all bricks are empty, the LBVH construction may result in an empty or trivial structure [G].
*   **Precision:** GPU-based construction requires careful handling of floating-point precision for bounding box calculations to avoid gaps or overlaps [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_lbvh_sparse_volume(bricks, brick_size):
    """
    Constructs an LBVH for sparse volumetric data.
    
    Args:
        bricks: List of tuples (x, y, z) representing grid coordinates of non-empty bricks.
        brick_size: Fixed size of each brick in the uniform grid.
        
    Returns:
        root: Root node of the constructed hierarchy.
    """
    # Step 1 & 2: Generate bounding boxes for non-empty bricks
    primitives = []
    for x, y, z in bricks:
        primitives.append({
            "min": (x * brick_size, y * brick_size, z * brick_size),
            "max": ((x + 1) * brick_size, (y + 1) * brick_size, (z + 1) * brick_size)
        })

    # Step 3: Construct LBVH
    # [G] Standard LBVH construction: sort primitives by Morton code,
    # then recursively split the list in the middle to form the hierarchy.
    # The sources do not provide the specific LBVH construction algorithm details.
    return _lbvh_build(primitives)

def _lbvh_build(primitives):
    if not primitives:
        return None
    if len(primitives) == 1:
        return {"bbox": primitives[0], "left": None, "right": None, "is_leaf": True}
    
    # [G] Assuming primitives are sorted by Morton code.
    # Standard implementation computes morton codes, sorts, and splits at midpoint.
    mid = len(primitives) // 2
    left_child = _lbvh_build(primitives[:mid])
    right_child = _lbvh_build(primitives[mid:])
    
    # Merge bounding boxes
    min_bbox = tuple(min(a, b) for a, b in zip(left_child["bbox"]["min"], right_child["bbox"]["min"]))
    max_bbox = tuple(max(a, b) for a, b in zip(left_child["bbox"]["max"], right_child["bbox"]["max"]))
    return {"bbox": {"min": min_bbox, "max": max_bbox}, "left": left_child, "right": right_child, "is_leaf": False}
```

## Subtopics

### [Linear Bounding Volume Hierarchy (LBVH) for Volume Rendering](11217-linear-bounding-volume-hierarchy-lbvh-for-volume-rendering.md)
Linear Bounding Volume Hierarchy (LBVH) is a hierarchical data structure for sparse volume rendering that enables empty space skipping, adapted from surface ray tracing to uniform grids and targeted for GPU implementation via CUDA. It sets itself apart from SAH-based methods (such as k-d trees) by using a middle-split heuristic on Morton-ordered bricks, offering significantly superior construction performance at the cost of inferior culling properties.

## Operating characteristics
*   **Performance:** Allows constructing an acceleration structure on the GPU at interactive frame rates for $1024^3$ volumes [S1, unofficial].
*   **Rendering Speed:** The LBVH structure allows for two to three times higher frame rates than using no empty-space skipping during ray marching [S1, unofficial].
*   **Scalability:** Designed for GPU implementation, suitable for large volumes [S1, unofficial].
*   **Limitations:** Not given in the sources.

## Background
Zellmann et al. [ZHL19] proposed this adaptation of a linear bounding volume hierarchy (LBVH) construction algorithm for triangle geometry to sparse volumetric data [S1, unofficial]. They used the boundaries of non-empty bricks as bounding geometry [S1, unofficial]. This work is part of a broader effort to support run-time re-construction of acceleration data structures for empty-space skipping in volume rendering [S1, unofficial].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
