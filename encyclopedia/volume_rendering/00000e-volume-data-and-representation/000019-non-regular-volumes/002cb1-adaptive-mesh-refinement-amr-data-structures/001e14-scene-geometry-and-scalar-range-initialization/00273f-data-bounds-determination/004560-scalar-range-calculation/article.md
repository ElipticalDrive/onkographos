# Scalar Range Calculation

Also called: Global Scalar Range Computation, Union of Point and Cell Ranges

## Overview
Computes the global minimum and maximum values for all scalar fields (both point and cell data) to establish the visualization scalar range. It solves the problem of determining the correct data bounds for color and opacity mapping by handling the union of potentially disjoint point and cell scalar ranges. It differs from [Local Density Bounds via Sliding Window](12750-local-density-bounds-via-sliding-window.md) by computing global extrema rather than local density statistics.

## How it works
**Inputs:** A mesh object containing `pointScalars`, `cellScalars`, and optionally a legacy `scalars` vector. An optional `scalarName` parameter to restrict calculation to a specific field.
**Outputs:** A global scalar range `[scalarMin, scalarMax]`.

**Data Structures:**
- `rangeMaps`: A map from scalar name to a tuple `(min, max)`.
- `scalarMin`, `scalarMax`: Float values representing the global bounds.

**Steps:**
1. Initialize `rangeMaps` as empty.
2. Iterate over `mesh.attributes->pointScalars`. For each scalar vector, compute its range using `FieldStore::computeRange` and store in `rangeMaps`.
3. Iterate over `mesh.attributes->cellScalars`. For each scalar vector, compute its range using `FieldStore::computeRange` and store in `rangeMaps`.
4. If `scalarName` is specified:
    a. Retrieve the point range for `scalarName` from `rangeMaps`.
    b. Retrieve the cell range for `scalarName` from `rangeMaps`.
    c. If both exist, set `scalarMin = min(pointMin, cellMin)` and `scalarMax = max(pointMax, cellMax)`.
    d. If only point range exists, use it.
    e. If only cell range exists, use it.
5. If `scalarName` is not specified or no specific scalar was found:
    a. If `mesh.attributes->pointScalars` is empty, use the range of `mesh.scalars` (legacy vector).
    b. Otherwise, compute the union of all ranges in `rangeMaps`: `scalarMin = min(all mins)`, `scalarMax = max(all maxs)`.

**Edge Cases:**
- Empty scalar arrays: Handled by checking existence before access.
- Disjoint point/cell data: The union operation ensures the full range is captured.
- Legacy data: Fallback to `mesh.scalars` if modern attributes are absent.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_scalar_range(mesh, scalar_name=None):
    range_maps = {}
    
    # Process point scalars
    if hasattr(mesh.attributes, 'pointScalars'):
        for name, vector in mesh.attributes.pointScalars.items():
            min_val, max_val = compute_range(vector)
            range_maps[f"point_{name}"] = (min_val, max_val)
            
    # Process cell scalars
    if hasattr(mesh.attributes, 'cellScalars'):
        for name, vector in mesh.attributes.cellScalars.items():
            min_val, max_val = compute_range(vector)
            range_maps[f"cell_{name}"] = (min_val, max_val)
            
    if scalar_name is not None:
        point_key = f"point_{scalar_name}"
        cell_key = f"cell_{scalar_name}"
        
        point_range = range_maps.get(point_key)
        cell_range = range_maps.get(cell_key)
        
        if point_range and cell_range:
            scalar_min = min(point_range[0], cell_range[0])
            scalar_max = max(point_range[1], cell_range[1])
        elif point_range:
            scalar_min, scalar_max = point_range
        elif cell_range:
            scalar_min, scalar_max = cell_range
        else:
            # Fallback if specific scalar not found in attributes
            if hasattr(mesh, 'scalars') and mesh.scalars:
                scalar_min, scalar_max = compute_range(mesh.scalars)
            else:
                scalar_min, scalar_max = 0.0, 1.0 # Default or error
    else:
        # Global range over all scalars
        if not range_maps:
            if hasattr(mesh, 'scalars') and mesh.scalars:
                scalar_min, scalar_max = compute_range(mesh.scalars)
            else:
                scalar_min, scalar_max = 0.0, 1.0
        else:
            all_mins = [r[0] for r in range_maps.values()]
            all_maxs = [r[1] for r in range_maps.values()]
            scalar_min = min(all_mins)
            scalar_max = max(all_maxs)
            
    return scalar_min, scalar_max

def compute_range(vector):
    # Standard min/max computation
    if not vector:
        return 0.0, 0.0
    return min(vector), max(vector)
```

## Subtopics

### MinMaxVoxel
A parallel implementation for OpenVDB leaf arrays. It uses TBB `parallel_reduce` to split the leaf array range across threads. Each thread computes local min/max, and results are merged via `std::min`/`std::max`. It initializes `mMin` to `numeric_limits::max()` and `mMax` to `numeric_limits::lowest()`. Fields: `LeafArray& mLeafArray`, `ValueType mMin`, `ValueType mMax`. [S2, unofficial]

## Operating characteristics
Time complexity is linear with respect to the number of scalar values $O(N)$. Memory complexity is $O(1)$ auxiliary space for the global range, or $O(K)$ for storing intermediate ranges if $K$ scalar fields exist. Performance scales linearly with parallelization in the MinMaxVoxel variant. Strengths: robust handling of disjoint point/cell data. Limitations: requires full data traversal. Use when global normalization is required. Avoid for streaming data where global bounds are unknown.

## Background
Originated in CFD visualization pipelines for handling complex mesh data structures with separate point and cell attributes. Developed as part of the SciRender project for scientific visualization. [S1, unofficial]

## Sources
[S1] thermeng/SciRender, https://github.com/thermeng/SciRender, fetched 2026-10-26 (unofficial)
[S2] AcademySoftwareFoundation/openvdb, https://github.com/AcademySoftwareFoundation/openvdb, fetched 2026-10-27 (unofficial)
