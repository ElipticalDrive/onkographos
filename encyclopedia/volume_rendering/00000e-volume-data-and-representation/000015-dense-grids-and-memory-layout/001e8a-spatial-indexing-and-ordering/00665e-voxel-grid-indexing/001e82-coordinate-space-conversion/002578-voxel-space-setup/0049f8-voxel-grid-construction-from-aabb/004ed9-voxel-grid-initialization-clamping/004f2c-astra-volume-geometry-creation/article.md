# ASTRA Volume Geometry Creation

Also called: Parallel Beam Volume Size Determination, Volume Geometry Creation, proj2vol

## Overview
ASTRA Volume Geometry Creation constructs a volume geometry dictionary that defines the grid dimensions and spatial bounding box for 2D or 3D volume reconstruction [S1, unofficial]. It solves the problem of mapping detector parameters (count and spacing) to a voxel grid's physical extent, ensuring the volume bounds encompass the entire projection data [S1, unofficial]. It differs from generic grid initialization by deriving spatial bounds directly from detector geometry or default centering rather than requiring explicit world coordinates [S1, unofficial].

## How it works
**Inputs:**
- `varargin`: Variable arguments specifying grid dimensions and optional spatial bounds. Accepted signatures include 1 integer (cubic volume dimension $V$), 1 tuple of 2 integers (2D: rows, cols), 1 tuple of 3 integers (3D: rows, cols, slices), 2 integers (2D: rows, cols), 6 arguments (2D: rows, cols, min_x, max_x, min_y, max_y), 3 integers (3D: rows, cols, slices), or 9 arguments (3D: rows, cols, slices, min_x, max_x, min_y, max_y, min_z, max_z) [S1, unofficial].
- `proj_geom`: Dictionary containing `'DetectorColCount'`, `'DetectorSpacingX'`, `'DetectorRowCount'`, and `'DetectorSpacingY'` used when deriving bounds from detector size [S1, unofficial].

**Outputs:**
- A dictionary representing the volume geometry with keys: `'GridRowCount'` (int), `'GridColCount'` (int), `'GridSliceCount'` (int, optional), and `'option'` (dict containing `'WindowMinX'`, `'WindowMaxX'`, `'WindowMinY'`, `'WindowMaxY'`, `'WindowMinZ'`, `'WindowMaxZ'` as floats) [S1, unofficial].

**Data Structures:**
The output is a flat dictionary where the `'option'` sub-dictionary stores six float values representing the axis-aligned bounding box (AABB) of the volume in world coordinates [S1, unofficial].

**Steps:**
1. Parse `varargin` to extract `GridRowCount` ($R$), `GridColCount` ($C$), and optionally `GridSliceCount` ($S$) [S1, unofficial].
2. Determine bounds strategy: If 6 or 9 arguments are provided, use the provided window bounds directly [S1, unofficial]. If bounds are not explicitly set, default to centering the volume at the origin [S1, unofficial].
3. Calculate default bounds: Set $\text{WindowMinX} = -C/2$, $\text{WindowMaxX} = C/2$, $\text{WindowMinY} = -R/2$, $\text{WindowMaxY} = R/2$, and for 3D: $\text{WindowMinZ} = -S/2$, $\text{WindowMaxZ} = S/2$ [S1, unofficial].
4. Calculate detector-based bounds (external usage pattern): Compute $max\_x = 1.0 \times (DetectorColCount \times DetectorSpacingX \times 0.5)$ and $max\_z = 1.0 \times (DetectorRowCount \times DetectorSpacingY \times 0.5)$ [S1, unofficial]. Set $max\_coord = \max(max\_x, max\_z)$ [S1, unofficial]. Set volume bounds to $[-max\_coord, max\_coord]$ for all axes and invoke `create_vol_geom(V, V, V, -max\_coord, max\_coord, -max\_coord, max\_coord, -max\_coord, max\_coord)` [S1, unofficial].
5. Construct geometry: Return the dictionary with grid dimensions and calculated window bounds [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_vol_geom(*args):
    num_args = len(args)
    if num_args == 1:
        V = args[0]
        R, C, S = V, V, V
        has_bounds = False
    elif num_args == 2:
        R, C = args
        S = None
        has_bounds = False
    elif num_args == 3:
        R, C, S = args
        has_bounds = False
    elif num_args == 6:
        R, C, min_x, max_x, min_y, max_y = args
        S = None
        has_bounds = True
    elif num_args == 9:
        R, C, S, min_x, max_x, min_y, max_y, min_z, max_z = args
        has_bounds = True
    else:
        raise ValueError("Unsupported number of arguments")
    
    option = {}
    if has_bounds:
        option['WindowMinX'] = min_x
        option['WindowMaxX'] = max_x
        option['WindowMinY'] = min_y
        option['WindowMaxY'] = max_y
        if S is not None:
            option['WindowMinZ'] = min_z
            option['WindowMaxZ'] = max_z
    else:
        option['WindowMinX'] = -C / 2.0
        option['WindowMaxX'] = C / 2.0
        option['WindowMinY'] = -R / 2.0
        option['WindowMaxY'] = R / 2.0
        if S is not None:
            option['WindowMinZ'] = -S / 2.0
            option['WindowMaxZ'] = S / 2.0
            
    geometry = {
        'GridRowCount': R,
        'GridColCount': C,
        'option': option
    }
    if S is not None:
        geometry['GridSliceCount'] = S
    return geometry
```

**Edge Cases and Pitfalls:**
- **3D Consistency:** If `GridSliceCount` is provided, all Z-axis bounds must be defined [S1, unofficial].
- **Precision:** Division by 2 uses floating-point arithmetic to avoid integer truncation errors [S1, unofficial].
- **Detector Mismatch:** If detector-based bounds are used, ensure `DetectorColCount` and `DetectorRowCount` correspond to the correct axes relative to `GridColCount` and `GridRowCount` [S1, unofficial].

## Subtopics

### ASTRA Geometry Conversion
Converts between ASTRA's dictionary-based geometry format and tomosipo's VolumeGeometry/ProjectionGeometry objects [S2, unofficial]. The `from_astra(astra_geom)` function validates input is a dict and routes to volume conversion if it contains 'GridSliceCount' [S2, unofficial]. It extracts WindowMinX/Y/Z and WindowMaxX/Y/Z from `astra_geom['option']` and extracts GridColCount, GridRowCount, GridSliceCount as voxX, voxY, voxZ [S2, unofficial]. It sets shape = (voxZ, voxY, voxX) and constructs extent = ((WindowMinZ, WindowMaxZ), (WindowMinY, WindowMaxY), (WindowMinX, WindowMaxX)) [S2, unofficial]. It calls `_extent_to_pos_size(extent)` to derive pos and size, then returns VolumeGeometry(shape=shape, pos=pos, size=size) [S2, unofficial]. The `to_astra(x)` function calls `x.to_astra()`, catching AttributeError and raising TypeError suggesting `ts.from_astra` [S2, unofficial]. The input `astra_geom` is a dictionary created by `astra.create_vol_geom` describing a 3D volume geometry [S2, unofficial]. The ASTRA volume geometry dict keys are 'option' (dict containing WindowMinX, WindowMaxX, WindowMinY, WindowMaxY, WindowMinZ, WindowMaxZ), 'GridSliceCount', 'GridRowCount', 'GridColCount' [S2, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] jakeoung/ShapeFromProjections, https://github.com/jakeoung/ShapeFromProjections, fetched 2026-10-26 (unofficial)
[S2] cicwi/tomosipo, https://github.com/cicwi/tomosipo, fetched 2026-10-26 (unofficial)
