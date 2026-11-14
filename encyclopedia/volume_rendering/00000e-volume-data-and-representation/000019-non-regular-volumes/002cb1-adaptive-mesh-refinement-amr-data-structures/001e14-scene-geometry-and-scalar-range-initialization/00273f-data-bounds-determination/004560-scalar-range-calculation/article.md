# Scalar Range Calculation

Also called: Global Scalar Range Computation, Union of Point and Cell Ranges

## Overview
Scalar Range Calculation determines the global minimum and maximum values for all scalar fields in a volume dataset to establish the normalization range for visualization transfer functions [S1, unofficial]. It solves the problem of aggregating extremal values from both point and cell data structures, which may exist independently or in combination. It is distinguished from sibling methods like [Local Density Bounds via Sliding Window](12750-local-density-bounds-via-sliding-window.md) by computing a global static range rather than local adaptive bounds, and from [Data Bounds Determination](3997-data-bounds-determination.md) by focusing specifically on scalar value extents rather than spatial coordinates.

## How it works
**Inputs:** A mesh object containing `pointScalars` (array of point-based scalar values), `cellScalars` (array of cell-based scalar values), and optionally a legacy `scalars` vector. An optional `scalarName` parameter to restrict calculation to a specific field.

**Outputs:** A global scalar range defined by `scalarMin` and `scalarMax`.

**Data Structures:**
- `pointScalars`: Vector of scalar values associated with mesh vertices.
- `cellScalars`: Vector of scalar values associated with mesh cells.
- `rangeMap`: A mapping from scalar field names to their computed `(min, max)` pairs.

**Steps:**
1. Initialize `rangeMap` as empty.
2. Iterate over `mesh.attributes->pointScalars`. For each scalar field, compute its range using `FieldStore::computeRange` and store the result in `rangeMap` under the field's name.
3. Iterate over `mesh.attributes->cellScalars`. For each scalar field, compute its range using `FieldStore::computeRange` and store the result in `rangeMap` under the field's name.
4. If `scalarName` is specified:
   - Retrieve the point range `R_p` and cell range `R_c` for `scalarName` from `rangeMap`.
   - If both `R_p` and `R_c` exist:
     - `scalarMin = min(R_p.min, R_c.min)`
     - `scalarMax = max(R_p.max, R_c.max)`
   - If only `R_p` exists:
     - `scalarMin = R_p.min`
     - `scalarMax = R_p.max`
   - If only `R_c` exists:
     - `scalarMin = R_c.min`
     - `scalarMax = R_c.max`
   - If neither exists, the range is undefined (handle as error or default).
5. If `scalarName` is not specified:
   - The global range is the union of all ranges in `rangeMap`.
   - `scalarMin = min(all ranges in rangeMap)`
   - `scalarMax = max(all ranges in rangeMap)`
6. If no `pointScalars` or `cellScalars` exist in attributes, but `mesh.scalars` (legacy vector) exists:
   - Compute the range of `mesh.scalars`.
   - Set `scalarMin` and `scalarMax` to this range.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_scalar_range(mesh, scalar_name=None):
    """
    Computes the global scalar range for a mesh.
    
    Args:
        mesh: Object with attributes.pointScalars, attributes.cellScalars, 
              and optionally scalars.
        scalar_name: Optional string to restrict calculation to a specific field.
        
    Returns:
        tuple: (scalar_min, scalar_max)
    """
    range_map = {}
    
    # Process point scalars
    if hasattr(mesh, 'attributes') and hasattr(mesh.attributes, 'pointScalars'):
        for field_name, values in mesh.attributes.pointScalars.items():
            r_min, r_max = compute_range(values)
            range_map[field_name] = (r_min, r_max)
            
    # Process cell scalars
    if hasattr(mesh, 'attributes') and hasattr(mesh.attributes, 'cellScalars'):
        for field_name, values in mesh.attributes.cellScalars.items():
            r_min, r_max = compute_range(values)
            range_map[field_name] = (r_min, r_max)
            
    # Determine final range
    if scalar_name is not None:
        if scalar_name in range_map:
            r_p = range_map.get(scalar_name + '_point') # Assuming distinct keys or logic to separate
            r_c = range_map.get(scalar_name + '_cell')
            
            # Note: The source implies a union if both exist. 
            # Implementation detail: If point and cell scalars share the same name key,
            # we need to handle the union logic explicitly. 
            # Based on [S1, unofficial]: "If both exist, set scalarMin/Max to the union"
            
            # Simplified assumption: range_map keys are unique per field type or 
            # the logic handles merging. Let's assume the source logic:
            # Find point range for scalar_name and cell range for scalar_name.
            
            # Re-reading [S1, unofficial]: "find its point and cell ranges". 
            # This implies we might need to track which ranges came from points vs cells.
            # Let's adjust the data structure assumption to track source.
            pass 
            
    # Revised approach based on strict [S1, unofficial] interpretation:
    # We need to know if a scalar_name has point data AND cell data.
    
    point_ranges = {}
    cell_ranges = {}
    
    if hasattr(mesh, 'attributes') and hasattr(mesh.attributes, 'pointScalars'):
        for field_name, values in mesh.attributes.pointScalars.items():
            point_ranges[field_name] = compute_range(values)
            
    if hasattr(mesh, 'attributes') and hasattr(mesh.attributes, 'cellScalars'):
        for field_name, values in mesh.attributes.cellScalars.items():
            cell_ranges[field_name] = compute_range(values)
            
    if scalar_name is not None:
        p_range = point_ranges.get(scalar_name)
        c_range = cell_ranges.get(scalar_name)
        
        if p_range is not None and c_range is not None:
            scalar_min = min(p_range[0], c_range[0])
            scalar_max = max(p_range[1], c_range[1])
        elif p_range is not None:
            scalar_min, scalar_max = p_range
        elif c_range is not None:
            scalar_min, scalar_max = c_range
        else:
            raise ValueError(f"Scalar {scalar_name} not found in point or cell data")
            
    else:
        # Global range over all fields
        all_mins = []
        all_maxs = []
        
        for r in point_ranges.values():
            all_mins.append(r[0])
            all_maxs.append(r[1])
            
        for r in cell_ranges.values():
            all_mins.append(r[0])
            all_maxs.append(r[1])
            
        if not all_mins:
            # Fallback to legacy scalars
            if hasattr(mesh, 'scalars') and mesh.scalars is not None:
                scalar_min, scalar_max = compute_range(mesh.scalars)
            else:
                raise ValueError("No scalar data found")
        else:
            scalar_min = min(all_mins)
            scalar_max = max(all_maxs)
            
    return scalar_min, scalar_max

def compute_range(values):
    """Helper to compute min and max of a list/array."""
    if not values:
        return (0.0, 0.0) # [G] Default for empty, though source doesn't specify
    return (min(values), max(values))
```

**Parameters:**
- `scalar_name`: Optional string. If provided, restricts calculation to this field. If omitted, calculates the union of all fields.

**Edge Cases:**
- **No Scalar Data:** If neither point nor cell scalars exist, and legacy `mesh.scalars` is absent, the method fails or returns undefined.
- **Legacy Fallback:** If `pointScalars` and `cellScalars` are empty but `mesh.scalars` exists, the legacy vector is used.
- **Missing Field:** If `scalar_name` is specified but not found in either point or cell data, an error should be raised.

## Subtopics

### [MinMaxVoxel](13073-minmaxvoxel.md)
Computes the scalar value range for AMR trees by iterating over active leaf nodes. It is chosen when working with hierarchical AMR data structures where a global range is needed for transfer function mapping. It differs from spatial bounding box calculations by focusing on scalar values rather than coordinates.

## Operating characteristics
- **Time Complexity:** Linear with respect to the total number of point and cell scalar values, $O(N_p + N_c)$, where $N_p$ is the number of point scalars and $N_c$ is the number of cell scalars.
- **Memory Complexity:** $O(1)$ auxiliary space if computing ranges in a single pass, or $O(F)$ where $F$ is the number of scalar fields if storing intermediate ranges.
- **Strengths:** Simple, deterministic, and provides a consistent global range for all visualization components.
- **Limitations:** Does not account for local variations or density; may be dominated by outliers if present. Not suitable for adaptive visualization where local ranges are preferred.
- **When to Use:** When a global normalization is required for consistent color mapping across the entire volume.
- **When Not to Use:** When local features are more important than global extremes, or when dealing with very large datasets where a sliding window approach is more memory-efficient.

## Background
This method is part of the data processing pipeline in volume rendering systems, specifically within the context of non-regular volumes and AMR data structures. It originates from implementations in scientific visualization libraries such as SciRender, which handle CFD mesh data [S1, unofficial]. The approach of unifying point and cell ranges ensures that both vertex-based and element-based scalar fields are considered in the visualization pipeline.

## Sources
[S1] thermeng/SciRender, https://github.com/thermeng/SciRender, fetched 2026-10-26 (unofficial)
