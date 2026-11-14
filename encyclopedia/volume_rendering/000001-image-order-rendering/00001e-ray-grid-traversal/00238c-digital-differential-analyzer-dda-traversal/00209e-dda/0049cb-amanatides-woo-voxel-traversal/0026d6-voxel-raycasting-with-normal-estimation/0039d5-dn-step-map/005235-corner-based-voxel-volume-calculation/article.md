# Corner-based Voxel Volume Calculation

Also called: Corner-based Voxel Volume Calculation for DDA Raymarching, corner-based volume, surface block maxing

## Overview
A technique to compute voxel volume values at corners for smooth terrain rendering, replacing side-based volume storage with corner-based values to better represent surface geometry during raymarching [S1, unofficial]. It sets apart from sibling methods by explicitly handling diagonal neighbors (NW, NE, SW, SE) and applying "surface block maxing" to clamp volumes at surface boundaries, whereas siblings like [Midpoint UV Calculation](14669-midpoint-uv-calculation.md) focus on interpolation precision or [Voxel DDA Raymarching with Adaptive Edge Smoothing](14664-voxel-dda-raymarching-with-adaptive-edge-smoothing.md) focus on edge artifacts without the specific corner-maxing logic.

## How it works
**Inputs:**
- `WX`: Current voxel x-coordinate index.
- `Walls`: Total width of the volume in voxels (dimension size).
- `Volume`: The 3D voxel grid data structure.
- `ColorIndex`: A value associated with each voxel indicating its material type (0 for air, non-zero for solid).
- `SurfaceBoxHeight`: The maximum height/depth value for a surface block.

**Outputs:**
- Updated `Volume` values at the current corner/voxel position.

**Data Structures:**
- `Volume[x][y][z]`: Stores the volume/height value.
- `ColorIndex[x][y][z]`: Stores the material identifier.

**Steps:**
1. **Boundary Check:** Verify if `WX == 0` or `WX == Walls - 1`. If true, skip complex neighbor averaging to prevent edge corruption at x=0 and x=63 (assuming a 64-width volume) [S1, unofficial].
2. **Non-Surface Tile Handling:** If the current voxel is not a surface tile (i.e., it is a full block), set its volume directly to the tile's volume. Do not average with neighbors [S1, unofficial].
3. **Surface Block Maxing:** If any adjacent voxel is a surface block, clamp or max out the volume to the `SurfaceBoxHeight` [S1, unofficial].
4. **Neighbor Summation:** Sum the `ColorIndex` values of the checked neighbors (Left, Right, Front, Back, and diagonals NW, NE, SW, SE) [S1, unofficial].
5. **Solid Check:** If the sum of `ColorIndex` values is greater than zero, set the volume content to 7. This indicates that at least one wall is not air [S1, unofficial].
6. **Air Handling:** If the sum is zero, set the volume content to 0 [S1, unofficial].
7. **Average Calculation:** If none of the sidewalls are surface blocks, calculate a float average: `(Sum of indices) * 0.5 / 4` [S1, unofficial].
8. **Shader Deferment:** Defer complex intersection depth calculations along angled lines to the shader, as CPU-side generation does not fully derive the exact mathematical intersection point [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_corner_volume(WX, Walls, Volume, ColorIndex, SurfaceBoxHeight):
    # Step 1: Boundary handling
    if WX == 0 or WX == Walls - 1:
        # Skip complex neighbor averaging to prevent edge corruption
        # [G] Implementation detail: Return current volume or handle edge case specifically
        return Volume[WX]

    # Step 2: Non-surface tile handling
    if not is_surface_tile(Volume[WX]):
        # Set volume directly to tile's volume (full block)
        Volume[WX] = get_tile_volume(Volume[WX])
        return Volume[WX]

    # Step 3: Surface block maxing
    # Check adjacent voxels (Left, Right, Front, Back, NW, NE, SW, SE)
    neighbors = get_neighbor_indices(WX)
    for neighbor in neighbors:
        if is_surface_block(Volume[neighbor]):
            # Clamp to surface box height
            Volume[WX] = max(Volume[WX], SurfaceBoxHeight)
            return Volume[WX]

    # Step 4 & 5: Sum color indices and check for solid
    color_sum = 0
    for neighbor in neighbors:
        color_sum += ColorIndex[neighbor]

    if color_sum > 0:
        # At least one wall is not air
        Volume[WX] = 7
    else:
        # All walls are air
        Volume[WX] = 0

    # Step 7: Average calculation if no surface blocks
    # Note: This step is mutually exclusive with Step 3 in the source logic
    # "if none of the sidewalls is not a surface" -> if no surface blocks
    if not any(is_surface_block(Volume[n]) for n in neighbors):
        # Multiply by 0.5 and divide by 4
        Volume[WX] = (color_sum * 0.5) / 4.0

    return Volume[WX]

def is_surface_tile(volume_data):
    # [G] Placeholder for specific tile type check
    return volume_data.type == "surface"

def is_surface_block(volume_data):
    # [G] Placeholder for surface block check
    return volume_data.is_surface

def get_neighbor_indices(WX):
    # [G] Returns indices for Left, Right, Front, Back, NW, NE, SW, SE
    # Exact neighbor coordinates depend on 3D grid layout
    return []

def get_tile_volume(volume_data):
    # [G] Returns the volume value of the tile
    return volume_data.volume
```

**Parameters:**
- `SurfaceBoxHeight`: Not given in the sources.
- `ColorIndex` values: 0 for air, non-zero for solid. Specific non-zero values are not given.

**Edge Cases and Pitfalls:**
- **Boundary Corruption:** Incorrect boundary handling at `WX == 0` and `WX == Walls - 1` will corrupt every wall at x=0 and x=63 [S1, unofficial].
- **Intersection Precision:** The exact mathematical intersection point along angled lines is not fully derived in the CPU code and requires shader-side differential or angle calculations [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Better representation of surface geometry during raymarching by using corner-based values [S1, unofficial].
- **Limitations:** Requires shader-side calculations for precise intersection points [S1, unofficial]. Boundary handling is critical and prone to corruption if not implemented correctly [S1, unofficial].

## Background
Origin: Unofficial video tutorial "Smooth terrain with voxel DDA raymarching" [S1, unofficial].
History: Developed as part of a voxel DDA raymarching implementation for smooth terrain rendering [S1, unofficial].
Usage: Used in voxel-based terrain rendering to improve surface smoothness and accuracy [S1, unofficial].

## Sources
[S1] Smooth terrain with voxel DDA raymarching, https://www.youtube.com/watch?v=CxoC0E7inUg, fetched 2026-10-25 (unofficial)
