# CT Slice Voxel Construction
Also called: Grid Cell Construction from CT Slices

## Overview
Constructs a 3D grid cell (voxel) from two adjacent 2D CT slices by calculating the 3D coordinates of its 8 vertices and retrieving the scalar values (Hounsfield units) for each vertex. It differs from [3D Coordinate Grid Generation](2333-3d-coordinate-grid-generation.md) by explicitly handling the discrete nature of medical imaging data, using slice-specific metadata (pixel spacing, origin) rather than a normalized [-1, 1] domain, and mapping values directly from the source image arrays.

## How it works
**Inputs:**
- `theSliceIndex`: Index of the front slice.
- `theRowIndex`: Row index in the 2D slice grid.
- `theColumnIndex`: Column index in the 2D slice grid.
- `CTSliceFront`: Metadata object for the front slice (Z-plane $Z_{front}$, origin $(X_{UL}, Y_{UL})$, pixel spacing $(S_x, S_y)$).
- `CTSliceBack`: Metadata object for the back slice (Z-plane $Z_{back}$, origin $(X_{UL}, Y_{UL})$, pixel spacing $(S_x, S_y)$).

**Outputs:**
- `GridCell`: A structure containing:
  - `p[0..7]`: Array of 8 `Point3D` objects representing the voxel vertices.
  - `val[0..7]`: Array of 8 integers representing the Hounsfield unit (HU) values at each vertex.

**Data Structures:**
- `Point3D`: Contains `x`, `y`, `z` (double).
- `CTSliceInfo`: Contains `UpperLeft_X`, `UpperLeft_Y`, `UpperLeft_Z`, `PixelSpacing_X`, `PixelSpacing_Y`.
- `GridCell`: Contains `p` (array of 8 `Point3D`) and `val` (array of 8 `Int32`).

**Vertex Convention:**
The vertices are mapped according to the Marching Cubes convention (Paul Bourke's index convention [S1, unofficial]):
- Indices 0-3: Back slice (Bottom-Left, Bottom-Right, Top-Right, Top-Left).
- Indices 4-7: Front slice (Bottom-Left, Bottom-Right, Top-Right, Top-Left).
*Note: The source code uses "Botton" (typo for Bottom) and "Top". The mapping is consistent with standard MC indexing where the first 4 vertices belong to the "back" (lower Z) and the last 4 to the "front" (higher Z), but the specific Y-ordering (Top/Bottom) must be verified against the coordinate system.*

**Steps:**
1. **Calculate X coordinates:**
   - Front Slice:
     - $X_{L,F} = X_{UL,F} + (\text{theColumnIndex} \times S_{x,F})$
     - $X_{R,F} = X_{L,F} + S_{x,F}$
   - Back Slice:
     - $X_{L,B} = X_{UL,B} + (\text{theColumnIndex} \times S_{x,B})$
     - $X_{R,B} = X_{L,B} + S_{x,B}$

2. **Calculate Y coordinates:**
   - Front Slice:
     - $Y_{T,F} = Y_{UL,F} + (\text{theRowIndex} \times S_{y,F})$
     - $Y_{B,F} = Y_{T,F} + S_{y,F}$
   - Back Slice:
     - $Y_{T,B} = Y_{UL,B} + (\text{theRowIndex} \times S_{y,B})$
     - $Y_{B,B} = Y_{T,B} + S_{y,B}$

3. **Calculate Z coordinates:**
   - $Z_F = Z_{UL,F}$
   - $Z_B = Z_{UL,B}$

4. **Assign vertex positions:**
   - $p[0] = (X_{L,B}, Y_{B,B}, Z_B)$
   - $p[1] = (X_{R,B}, Y_{B,B}, Z_B)$
   - $p[2] = (X_{R,F}, Y_{B,F}, Z_F)$
   - $p[3] = (X_{L,F}, Y_{B,F}, Z_F)$
   - $p[4] = (X_{L,B}, Y_{T,B}, Z_B)$
   - $p[5] = (X_{R,B}, Y_{T,B}, Z_B)$
   - $p[6] = (X_{R,F}, Y_{T,F}, Z_F)$
   - $p[7] = (X_{L,F}, Y_{T,F}, Z_F)$

5. **Retrieve scalar values (Hounsfield Units):**
   The source code implies a specific row/column indexing for the pixel array. Based on the provided C# snippet:
   - Back Slice (`val[0..5]`):
     - `val[0]` at `(theRowIndex + 1, theColumnIndex)`
     - `val[1]` at `(theRowIndex + 1, theColumnIndex + 1)`
     - `val[4]` at `(theRowIndex, theColumnIndex)`
     - `val[5]` at `(theRowIndex, theColumnIndex + 1)`
   - Front Slice (`val[2..7]`):
     - `val[2]` at `(theRowIndex + 1, theColumnIndex + 1)`
     - `val[3]` at `(theRowIndex + 1, theColumnIndex)`
     - `val[6]` at `(theRowIndex, theColumnIndex + 1)`
     - `val[7]` at `(theRowIndex, theColumnIndex)`

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_grid_cell(the_slice_index, the_row_index, the_col_index, slice_front, slice_back):
    """
    Constructs a GridCell from two adjacent CT slices.
    
    Args:
        the_slice_index: Index of the front slice.
        the_row_index: Row index in the 2D slice grid.
        the_col_index: Column index in the 2D slice grid.
        slice_front: Object with attributes UpperLeft_X, UpperLeft_Y, UpperLeft_Z, 
                     PixelSpacing_X, PixelSpacing_Y, and a method GetHounsfieldPixelValue(row, col).
        slice_back: Same structure as slice_front.
        
    Returns:
        A GridCell object with p[0..7] (Point3D) and val[0..7] (int).
    """
    
    # 1. Calculate X coordinates
    x_l_front = slice_front.UpperLeft_X + (the_col_index * slice_front.PixelSpacing_X)
    x_r_front = x_l_front + slice_front.PixelSpacing_X
    
    x_l_back = slice_back.UpperLeft_X + (the_col_index * slice_back.PixelSpacing_X)
    x_r_back = x_l_back + slice_back.PixelSpacing_X
    
    # 2. Calculate Y coordinates
    y_t_front = slice_front.UpperLeft_Y + (the_row_index * slice_front.PixelSpacing_Y)
    y_b_front = y_t_front + slice_front.PixelSpacing_Y
    
    y_t_back = slice_back.UpperLeft_Y + (the_row_index * slice_back.PixelSpacing_Y)
    y_b_back = y_t_back + slice_back.PixelSpacing_Y
    
    # 3. Calculate Z coordinates
    z_front = slice_front.UpperLeft_Z
    z_back = slice_back.UpperLeft_Z
    
    # 4. Assign vertex positions
    # p[0..3] are Back slice, p[4..7] are Front slice
    # Mapping based on source code:
    # 0: Back-Bottom-Left, 1: Back-Bottom-Right
    # 2: Front-Bottom-Right, 3: Front-Bottom-Left
    # 4: Back-Top-Left, 5: Back-Top-Right
    # 6: Front-Top-Right, 7: Front-Top-Left
    
    p = [None] * 8
    p[0] = Point3D(x_l_back, y_b_back, z_back)
    p[1] = Point3D(x_r_back, y_b_back, z_back)
    p[2] = Point3D(x_r_front, y_b_front, z_front)
    p[3] = Point3D(x_l_front, y_b_front, z_front)
    p[4] = Point3D(x_l_back, y_t_back, z_back)
    p[5] = Point3D(x_r_back, y_t_back, z_back)
    p[6] = Point3D(x_r_front, y_t_front, z_front)
    p[7] = Point3D(x_l_front, y_t_front, z_front)
    
    # 5. Retrieve scalar values
    # Note: The source code uses (row + 1) for "bottom" and (row) for "top".
    # This implies the coordinate system's Y-axis increases upwards, but the 
    # image array's row 0 is at the top (Y_max) or bottom (Y_min)?
    # The source code explicitly maps:
    # val[0] (Back-Bottom-Left) -> (row+1, col)
    # val[4] (Back-Top-Left) -> (row, col)
    # This suggests that in the image array, higher row index = lower Y coordinate (Bottom).
    
    val = [None] * 8
    
    # Back Slice Values
    val[0] = slice_back.GetHounsfieldPixelValue(the_row_index + 1, the_col_index)
    val[1] = slice_back.GetHounsfieldPixelValue(the_row_index + 1, the_col_index + 1)
    val[4] = slice_back.GetHounsfieldPixelValue(the_row_index, the_col_index)
    val[5] = slice_back.GetHounsfieldPixelValue(the_row_index, the_col_index + 1)
    
    # Front Slice Values
    val[2] = slice_front.GetHounsfieldPixelValue(the_row_index + 1, the_col_index + 1)
    val[3] = slice_front.GetHounsfieldPixelValue(the_row_index + 1, the_col_index)
    val[6] = slice_front.GetHounsfieldPixelValue(the_row_index, the_col_index + 1)
    val[7] = slice_front.GetHounsfieldPixelValue(the_row_index, the_col_index)
    
    return GridCell(p, val)

class Point3D:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

class GridCell:
    def __init__(self, p, val):
        self.p = p
        self.val = val
```

**Edge Cases and Pitfalls:**
- **Index Offset:** The code uses `theRowIndex + 1` for the bottom row values and `theRowIndex` for the top row values. This implies a specific coordinate system where row 0 might be the top or bottom. This must be consistent with the slice data layout [S1, unofficial]. If the image array is stored with row 0 at the top (standard image format), then `row + 1` corresponds to a lower Y coordinate (if Y increases upwards) or higher Y coordinate (if Y increases downwards). The source code's mapping of `val[0]` (Back-Bottom-Left) to `(row+1, col)` suggests that in the image array, a higher row index corresponds to the "Bottom" of the voxel in world space. This is a critical assumption that must be verified against the specific DICOM header or image storage format.
- **Boundary Conditions:** The algorithm assumes `theRowIndex + 1` and `theColumnIndex + 1` are valid indices. For the last row or column of a slice, this will cause an out-of-bounds error. The caller must ensure that the loop bounds exclude the last row and column, or handle boundary conditions explicitly.
- **Pixel Spacing:** The algorithm assumes uniform pixel spacing within a slice. If pixel spacing varies, the calculation must be adjusted.
- **Z-Ordering:** The algorithm assumes `slice_back` has a lower Z value than `slice_front`. If the slices are not ordered correctly, the voxel geometry will be inverted.

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel, as it involves a fixed number of arithmetic operations and array lookups.
- **Memory Complexity:** $O(1)$ per voxel, storing 8 points and 8 values.
- **Strengths:** Simple, efficient, and directly maps to the Marching Cubes algorithm's requirements.
- **Limitations:** Requires careful handling of image array indexing and coordinate system orientation. Not suitable for non-uniform voxel sizes or irregular grids.

## Background
This method is derived from a C# implementation for generating 3D meshes from CT slices using the Marching Cubes algorithm [S1, unofficial]. It is based on the vertex indexing convention described by Paul Bourke in "Polygonising a scalar field" [S1, unofficial]. The method is commonly used in medical imaging visualization to convert 2D CT scan slices into a 3D volumetric representation.

## Sources
[S1] How to generate 3D mesh from CT Slices (.raw) using marching cubes algorithm in C#?, https://stackoverflow.com/questions/65498236/how-to-generate-3d-mesh-from-ct-slices-raw-using-marching-cubes-algorithm-in, fetched 2026-10-27 (unofficial)
