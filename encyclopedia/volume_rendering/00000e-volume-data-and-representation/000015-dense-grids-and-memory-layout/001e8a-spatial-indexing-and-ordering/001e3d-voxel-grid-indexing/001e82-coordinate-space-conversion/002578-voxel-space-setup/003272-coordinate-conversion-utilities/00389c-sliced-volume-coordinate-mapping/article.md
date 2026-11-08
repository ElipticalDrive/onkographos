# Sliced Volume Coordinate Mapping

Also called: 3D to 2D Tiled Texture Coordinate Conversion, XYZ to XY Tile Mapping

## Overview
Sliced Volume Coordinate Mapping converts 3D voxel coordinates into 2D texture coordinates for volumetric data stored as a tiled grid of 2D slices within a single 2D image [S1, unofficial]. It solves the problem of accessing 3D volumetric data in raster-based renderers that lack native 3D texture sampling support by flattening the volume into a 2D layout [S1, unofficial]. It differs from standard 3D texture sampling by requiring explicit coordinate transformation logic to resolve 3D positions to 2D image pixels, and it uses linear interpolation between adjacent slices to reduce banding artifacts compared to nearest-neighbor sampling [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Continuous 3D voxel coordinates $(x, y, z)$ normalized to $[0, 1]$; total number of slices $ZSlices$; number of tile columns $TileColumns$; number of rows per tile $NumRows$.
*   **Outputs:** Two 2D coordinate vectors $(u_1, v_1, 0)$ and $(u_2, v_2, 0)$ corresponding to the two adjacent slices, and an interpolation weight $w$.

**Data Structures:**
No complex data structures are required. The method relies on scalar arithmetic and integer indexing. The volume is assumed to be stored in a 2D image where tiles are arranged in a grid of $TileColumns \times TileRows$. Each tile contains a 2D slice of the volume.

**Parameters:**
*   $ZSlices$: Total number of slices in the volume (e.g., 128 or 361) [S1, unofficial].
*   $TileColumns$: Number of tiles horizontally in the 2D image [S1, unofficial].
*   $NumRows$: Number of rows of pixels in each tile [S1, unofficial].

**Steps:**
1.  **Clamp and Scale Z:** Clamp the input $z$ coordinate to $[0, 1]$ and scale it by $ZSlices$ to get a continuous slice index $s$.
    $$ s = \min(1, \max(z, 0)) \times ZSlices $$
    This prevents out-of-bounds access [S1, unofficial].

2.  **Determine Slice Indices:** Calculate the integer indices for the two adjacent slices to interpolate between.
    $$ s_1 = \lfloor s \rfloor $$
    $$ s_2 = s_1 + 1 $$

3.  **Calculate Tile Coordinates for Slice 1:** Determine the tile position $(tx_1, ty_1)$ in the 2D image grid for slice $s_1$.
    $$ tx_1 = s_1 \pmod{TileColumns} $$
    $$ ty_1 = \lfloor \frac{s_1}{TileColumns} + 0.001 \rfloor $$
    The addition of $0.001$ to the division result before flooring avoids rounding crossover issues at tile boundaries [S1, unofficial].

4.  **Calculate Normalized UV for Slice 1:** Compute the normalized texture coordinates $(u_1, v_1)$ within the 2D image.
    $$ u_1 = \frac{\text{clip}(x) + tx_1}{TileColumns} $$
    $$ v_1 = \frac{\text{clip}(y) + ty_1}{NumRows} $$
    Here, $\text{clip}(x)$ and $\text{clip}(y)$ refer to clamping $x$ and $y$ to $[0, 1]$ [S1, unofficial].

5.  **Calculate Tile Coordinates for Slice 2:** Repeat for slice $s_2$.
    $$ tx_2 = s_2 \pmod{TileColumns} $$
    $$ ty_2 = \lfloor \frac{s_2}{TileColumns} + 0.001 \rfloor $$

6.  **Calculate Normalized UV for Slice 2:**
    $$ u_2 = \frac{\text{clip}(x) + tx_2}{TileColumns} $$
    $$ v_2 = \frac{\text{clip}(y) + ty_2}{NumRows} $$

7.  **Output Coordinates:** The method outputs two 2D coordinate vectors for texture sampling.
    $$ \text{Output}_1 = (u_1, v_1, 0) $$
    $$ \text{Output}_2 = (u_2, v_2, 0) $$

8.  **Calculate Interpolation Weight:** Determine the fractional part of the slice index for linear interpolation.
    $$ w = s - s_1 $$

**Pitfalls:**
*   **Rounding Crossover:** Without the $0.001$ offset in the tile row calculation, floating-point precision errors can cause incorrect tile selection at boundaries [S1, unofficial].
*   **Out-of-Bounds:** Failing to clamp $z$ before scaling can lead to accessing invalid slices [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sliced_volume_coordinate_mapping(x, y, z, ZSlices, TileColumns, NumRows):
    # Clamp z to [0, 1] and scale to slice index
    z_clamped = max(0.0, min(1.0, z))
    slice_index = z_clamped * ZSlices
    
    # Integer slice indices
    slice1 = int(slice_index)
    slice2 = slice1 + 1
    
    # Tile positions for slice 1
    tile_x1 = slice1 % TileColumns
    tile_y1 = int(slice1 / TileColumns + 0.001)
    
    # Normalized UV for slice 1
    x_clamped = max(0.0, min(1.0, x))
    y_clamped = max(0.0, min(1.0, y))
    
    u1 = (x_clamped + tile_x1) / TileColumns
    v1 = (y_clamped + tile_y1) / NumRows
    
    # Tile positions for slice 2
    tile_x2 = slice2 % TileColumns
    tile_y2 = int(slice2 / TileColumns + 0.001)
    
    # Normalized UV for slice 2
    u2 = (x_clamped + tile_x2) / TileColumns
    v2 = (y_clamped + tile_y2) / NumRows
    
    # Interpolation weight
    weight = slice_index - slice1
    
    return (u1, v1, 0), (u2, v2, 0), weight
```

## Subtopics

### [Voxel-to-Image Slicing (Tiled EXR)](8289-voxel-to-image-slicing-tiled-exr.md)
A specific application of this mapping in Blender's Cycles renderer, where voxel data is converted to a tiled EXR image. It differs by defining the specific image format and renderer context. Choose it when rendering 3D voxel data in Cycles without native 3D texture support.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample, involving basic arithmetic operations.
*   **Memory Complexity:** $O(1)$ auxiliary memory.
*   **Performance:** The method is computationally inexpensive, allowing for real-time or near-real-time evaluation in shader networks [S1, unofficial].
*   **Strengths:** Enables 3D volume rendering in 2D-only texture sampling environments; linear interpolation reduces banding artifacts compared to nearest-neighbor [S1, unofficial].
*   **Limitations:** Requires manual setup of node groups or scripts; precision issues at tile boundaries if not handled correctly (mitigated by the 0.001 offset) [S1, unofficial].
*   **When to Use:** When rendering volumetric data in raster-based renderers that do not support 3D textures, such as Cycles with tiled EXR inputs [S1, unofficial].
*   **When Not to Use:** When native 3D texture sampling is available, as it is more direct and less prone to coordinate mapping errors.

## Background
The method originated from community solutions for rendering volumetric data in Blender's Cycles renderer, specifically when using tiled EXR images generated from smoke simulations [S1, unofficial]. It was developed to overcome the lack of native 3D texture support in Cycles by flattening the volume into a 2D grid of slices [S1, unofficial]. The technique is implemented using the Node Expressions add-on or manually built node groups in Blender [S1, unofficial].

## Sources
[S1] Voxel data in Cycles, https://blender.stackexchange.com/questions/14995, fetched 2026-10-30 (unofficial)
