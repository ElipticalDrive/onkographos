# SDF Value Computation

Also called: IsoVal.compute

## Overview
SDF Value Computation is the core kernel that calculates the Signed Distance Function (SDF) value for each voxel in a grid. It transforms grid coordinates to world space, evaluates the SDF function, and stores the result. It builds on [Batched SDF Inference](2329-batched-sdf-inference.md) by performing the actual geometric evaluation for a batch of samples.

## How it works
**Inputs:**
- Grid ID (integer index of the voxel).
- `_ShapeToWorld`: A 4x4 matrix transforming local shape coordinates to world space.
- `SHAPE_SIZE`: Parameter defining the scale or bounds of the primitive shape.
- `FUNCTION`: The specific SDF primitive function to evaluate.

**Outputs:**
- SDF value for the voxel.

**Data Structures:**
- `_IsoVals`: Array storing computed SDF values.
- `_ShapeToWorld`: 4x4 transformation matrix.

**Steps:**
1. Get the surface position from the grid ID using `GetPosSurface(id)`.
2. Transform the position to world space using `_ShapeToWorld`.
3. Evaluate the SDF function (`FUNCTION`) at the world position with `SHAPE_SIZE`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sdf_value(id, shape_to_world, shape_size, function):
    # Step 1: Get surface position from grid ID
    local_pos = get_pos_surface(id)
    
    # Step 2: Transform to world space
    world_pos = transform_point(shape_to_world, local_pos)
    
    # Step 3: Evaluate SDF function
    sdf_value = function(world_pos, shape_size)
    
    return sdf_value
```

**Parameters:**
- `_ShapeToWorld`: A 4x4 matrix transforming local shape coordinates to world space.
- `SHAPE_SIZE`: Parameter defining the scale or bounds of the primitive shape.

**Edge Cases and Pitfalls:**
- Precision issues may arise if the transformation matrix is not properly normalized.
- Boundary conditions for the grid must be handled to avoid out-of-bounds access.

## Subtopics

## Operating characteristics
- **Time Complexity:** O(1) per voxel, as it involves a fixed number of transformations and function evaluations.
- **Memory Complexity:** O(N) for storing SDF values, where N is the number of voxels.
- **Strengths:** Efficient for large grids due to batched processing.
- **Limitations:** Performance may degrade with complex SDF functions or large transformation matrices.

## Background
Originated from the need to efficiently compute SDF values for voxel-based rendering. The method is part of a broader pipeline for implicit surface rendering.

## Sources
[S1] vinceniko/Isosurface, https://github.com/vinceniko/Isosurface, fetched 2026-10-27 (unofficial)
