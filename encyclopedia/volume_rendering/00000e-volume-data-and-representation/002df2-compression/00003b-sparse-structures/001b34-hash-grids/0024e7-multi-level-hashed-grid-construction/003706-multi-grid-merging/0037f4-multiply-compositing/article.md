# Multiply Compositing


## Overview
Multiply Compositing merges two sparse hash grids by taking the union of their active voxels, where overlapping values are multiplied together [S1, unofficial]. It differs from [Replace Compositing](7905-multi-grid-merging.md) by preserving both contributions in the overlap rather than letting one overwrite the other, making it suitable for additive or multiplicative blending of volumetric data.

## How it works
**Inputs:** Two sparse hash grids, `a` and `b`, containing active voxels with associated values.
**Outputs:** A new sparse hash grid, `data`, containing the merged result.

**Data Structures:**
The output `data` is a hash map (dictionary) mapping voxel coordinates `c` to their scalar values `v`.

**Steps:**
1. Initialize an empty hash map `data`.
2. Copy all active voxels from grid `a` into `data`. For each active voxel in `a` with coordinate `c` and value `v_a`, set `data[c] = v_a`.
3. Iterate through all active voxels in grid `b`. For each active voxel in `b` with coordinate `c` and value `v_b`:
   - If `c` exists in `data`, update the value: `data[c] = data[c] * v_b`.
   - If `c` does not exist in `data`, insert the value: `data[c] = v_b`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multiply_compositing(grid_a, grid_b):
    # Initialize empty hash map for the result
    data = {}
    
    # Step 1: Copy all active voxels from grid a
    for c, v_a in grid_a.active_voxels():
        data[c] = v_a
        
    # Step 2: Merge active voxels from grid b
    for c, v_b in grid_b.active_voxels():
        if c in data:
            # Overlapping voxel: multiply values
            data[c] = data[c] * v_b
        else:
            # Non-overlapping voxel: insert value
            data[c] = v_b
            
    return data
```

**Edge Cases and Pitfalls:**
- **Zero Values:** If any voxel value is zero, the product becomes zero. This may effectively remove density or color contributions in overlapping regions.
- **Precision:** Multiplying many small values can lead to underflow. Multiplying large values can lead to overflow. [G] Ensure values are normalized or clamped if necessary for the specific application.
- **Order Independence:** Multiplication is commutative, so the order of `a` and `b` does not affect the final result for overlapping voxels. However, the implementation must handle the case where `a` and `b` are the same grid correctly (squaring the values).

## Subtopics

### Replace Compositing
Returns a new grid starting with `a`'s active voxels, then overwriting with `b`'s active voxels. Grid `b` stamps on top of `a` [S1, unofficial].
**Steps:**
1. Copy all active voxels from grid `a` to `data`.
2. Iterate active voxels from grid `b`. Set `data[c] = v` (overwriting if present).
3. Build grid using `build_grid(data, a.tree.background; ...)` [S1, unofficial].
**When to choose:** Use when `b` represents a later stage or occluding volume that should completely replace `a`'s contribution in overlapping regions.

## Operating characteristics
**Time Complexity:** $O(N_a + N_b)$, where $N_a$ and $N_b$ are the number of active voxels in grids `a` and `b` respectively. Each voxel is visited once.
**Memory Complexity:** $O(N_a + N_b)$ in the worst case (no overlap), or $O(N_{union})$ where $N_{union}$ is the number of unique active voxel coordinates in the union of `a` and `b`.
**Strengths:** Simple, fast, and preserves information from both grids in overlapping regions.
**Limitations:** Multiplication can lead to rapid decay or growth of values, potentially requiring normalization. Not suitable for alpha blending where transparency needs to be composited additively.
**When to use:** When combining volumetric effects that are multiplicative (e.g., density accumulation in some physical models, or multiplicative color blending).
**When not to use:** When one volume should occlude another (use Replace Compositing) or when additive blending is required (use Add Compositing, if available).

## Background
Multiply Compositing is a method for merging sparse volumetric data structures, specifically hash grids, as part of the Multi-Grid Merging process in the Lyr.jl volume rendering framework [S1, unofficial]. It is used to combine multiple levels or sources of volumetric data into a single coherent representation.

## Sources
[S1] Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
