# XYZI+CUBE Format

Also called: CUBE chunk format, Optimised vox format

## Overview
The XYZI+CUBE format is an optional, optimized variant of the standard XYZI voxel storage format designed to reduce disk space by grouping contiguous voxels of the same color index into rectangular cuboid chunks [S1, unofficial]. It solves the inefficiency of storing individual voxel coordinates and indices for solid volumetric blocks by replacing them with a single bounding box definition [S1, unofficial]. Unlike its sibling [Voxel Chunk Decompression](6795-voxel-chunk-decompression.md), which handles the reading process, or [Bounding Volume Lossless (BVL) Compression](18657-bounding-volume-lossless-bvl-compression.md), which uses hierarchical bounds, XYZI+CUBE relies on a greedy expansion algorithm to identify maximal axis-aligned cubes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid represented as a matrix `matrix[x][y][z]` where each cell contains a color index (integer). The grid dimensions are `size.x`, `size.y`, `size.z`.
*   **Output:** A list of CUBE chunks. Each chunk is defined by its bounds and color index.

**Data Structures:**
*   **Matrix:** A 3D array `matrix[x][y][z]` storing the color index of each voxel. This structure is required for $O(1)$ access to adjacent voxels during the expansion phase [S1, unofficial].
*   **CUBE Chunk:** A record containing:
    *   `xs, ys, zs`: Start coordinates (inclusive) of the cube.
    *   `xe, ye, ze`: End coordinates (inclusive) of the cube.
    *   `index`: The color index shared by all voxels within the bounds.
    *   *Layout:* `Format(xs, ys, zs, xe, ye, ze, index)` [S1, unofficial].

**Algorithm Steps:**
1.  Initialize an empty list `cubes`.
2.  Iterate through every voxel location $(xx, yy, zz)$ in the grid, ordered from $(0,0,0)$ to $(size.x-1, size.y-1, size.z-1)$.
3.  Check the value `i = matrix[xx][yy][zz]`.
4.  If `i != 0` (a voxel exists):
    a.  Initialize a cube with `low = (xx, yy, zz)` and `high = (xx, yy, zz)`.
    b.  Initialize expansion flags `cubeExpanded = [true, true, true]` corresponding to X, Y, and Z axes.
    c.  **Expand X:** While `cubeExpanded[0]` is true:
        i.  Check if `high.x + 1 < size.x`.
        ii. If within bounds, verify that all voxels in the new X-face match index `i`. The new face consists of voxels at `x = high.x + 1` for all `y` in `[low.y, high.y]` and `z` in `[low.z, high.z]`.
        iii. If all match, set `high.x += 1`. Otherwise, set `cubeExpanded[0] = false`.
    d.  **Expand Y:** While `cubeExpanded[1]` is true:
        i.  Check if `high.y + 1 < size.y`.
        ii. If within bounds, verify that all voxels in the new Y-face match index `i`. The new face consists of voxels at `y = high.y + 1` for all `x` in `[low.x, high.x]` and `z` in `[low.z, high.z]`.
        iii. If all match, set `high.y += 1`. Otherwise, set `cubeExpanded[1] = false`.
    e.  **Expand Z:** While `cubeExpanded[2]` is true:
        i.  Check if `high.z + 1 < size.z`.
        ii. If within bounds, verify that all voxels in the new Z-face match index `i`. The new face consists of voxels at `z = high.z + 1` for all `x` in `[low.x, high.x]` and `y` in `[low.y, high.y]`.
        iii. If all match, set `high.z += 1`. Otherwise, set `cubeExpanded[2] = false`.
    f.  Append the resulting cube `(low, high, i)` to `cubes`.
    g.  **Mark Processed:** Iterate through all voxels $(x, y, z)$ where $x \in [low.x, high.x]$, $y \in [low.y, high.y]$, and $z \in [low.z, high.z]$. Set `matrix[x][y][z] = 0` to prevent reprocessing these voxels in subsequent iterations [S1, unofficial].
5.  Return `cubes`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_xyzicube(matrix, size):
    cubes = []
    # Iterate through every possible voxel location
    for xx in range(size.x):
        for yy in range(size.y):
            for zz in range(size.z):
                i = matrix[xx][yy][zz]
                
                # If a voxel exists (i != 0), initialize a cube
                if i != 0:
                    low = (xx, yy, zz)
                    high = (xx, yy, zz)
                    
                    # Flags for expansion in x, y, z directions
                    expand_x = True
                    expand_y = True
                    expand_z = True
                    
                    # Expand in X direction
                    while expand_x:
                        if high[0] + 1 < size.x:
                            # Check if the new X-face matches index i
                            # New face: x = high[0] + 1, y in [low[1]..high[1]], z in [low[2]..high[2]]
                            match = True
                            for y in range(low[1], high[1] + 1):
                                for z in range(low[2], high[2] + 1):
                                    if matrix[high[0] + 1][y][z] != i:
                                        match = False
                                        break
                                if not match:
                                    break
                            if match:
                                high = (high[0] + 1, high[1], high[2])
                            else:
                                expand_x = False
                        else:
                            expand_x = False

                    # Expand in Y direction
                    while expand_y:
                        if high[1] + 1 < size.y:
                            # Check if the new Y-face matches index i
                            # New face: y = high[1] + 1, x in [low[0]..high[0]], z in [low[2]..high[2]]
                            match = True
                            for x in range(low[0], high[0] + 1):
                                for z in range(low[2], high[2] + 1):
                                    if matrix[x][high[1] + 1][z] != i:
                                        match = False
                                        break
                                if not match:
                                    break
                            if match:
                                high = (high[0], high[1] + 1, high[2])
                            else:
                                expand_y = False
                        else:
                            expand_y = False

                    # Expand in Z direction
                    while expand_z:
                        if high[2] + 1 < size.z:
                            # Check if the new Z-face matches index i
                            # New face: z = high[2] + 1, x in [low[0]..high[0]], y in [low[1]..high[1]]
                            match = True
                            for x in range(low[0], high[0] + 1):
                                for y in range(low[1], high[1] + 1):
                                    if matrix[x][y][high[2] + 1] != i:
                                        match = False
                                        break
                                if not match:
                                    break
                            if match:
                                high = (high[0], high[1], high[2] + 1)
                            else:
                                expand_z = False
                        else:
                            expand_z = False

                    # Add the resulting cube to the list
                    cubes.append({
                        'xs': low[0], 'ys': low[1], 'zs': low[2],
                        'xe': high[0], 'ye': high[1], 'ze': high[2],
                        'index': i
                    })

                    # Mark all voxels in the cube as 0 to avoid reprocessing
                    for x in range(low[0], high[0] + 1):
                        for y in range(low[1], high[1] + 1):
                            for z in range(low[2], high[2] + 1):
                                matrix[x][y][z] = 0
                                
    return cubes
```

**Parameters and Edge Cases:**
*   **Color Index 0:** Typically represents empty space. Voxels with index 0 are skipped [S1, unofficial].
*   **Boundary Checks:** Expansion stops immediately if the next coordinate equals the grid dimension size [S1, unofficial].
*   **Degenerate Input:** If the input contains no voxels, the output is an empty list. If the input is a single voxel, the output is a cube with `low == high` [S1, unofficial].

## Subtopics

### [LINE and CHECK Chunks](20151-line-and-check-chunks.md)
LINE and CHECK chunks are optional compression primitives for the XYZI+CUBE format that handle specific geometric patterns—45-degree lines and checkerboards—respectively, which standard cubes cannot compress efficiently. They are distinct from the primary CUBE primitive by targeting linear and alternating spatial distributions rather than solid volumetric blocks.

### VMAP and CMAP Color Mapping
VMAP and CMAP are techniques to store color mapping more efficiently by sorting voxels/cubes by color and using separate chunks to map colors [S1, unofficial].
*   **Definition:** A technique to store colour mapping more efficiently by sorting voxels/cubes by colour and using separate chunks (VMAP for voxels, CMAP for cubes) to map colours [S1, unofficial].
*   **Step:** Sort voxels/cubes by colour. In the voxel chunk, store voxels in the same order as VMAP. If VMAP starts with N white voxels, the first N voxels are white by default, allowing storage of only location (3 bytes per voxel instead of 4) [S1, unofficial].
*   **Layout:** VMAP/CMAP chunk: 4 bytes per colour. Byte 0 is the colour index. Bytes 1-3 are the amount of voxels/cubes that have this colour [S1, unofficial].
*   **Pitfall:** This method can remove the need for the RGBA palette chunk by writing full colour (4 bytes) in the maps, removing the 255 colour limitation [S1, unofficial].
*   **Complexity:** Sorting voxals/cubes by colour is pretty fast, so the performance loss is negligible [S1, unofficial].
*   **Numbers:** Example voxels: 346 KB (354,654 bytes) in VMAP+XYZ+CMAP+CUBE format (down from 401 KB XYZI+CUBE). Custom voxels: 193 KB (197,934 bytes) in VMAP+XYZ+CMAP+CUBE format (down from 221 KB XYZI+CUBE) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Encoding performance is fast; the cube finding algorithm can read, find cubes, and write files in under 200ms for tested models [S1, unofficial]. However, encoding time is increased compared to raw XYZI because it requires finding cubes [S1, unofficial].
*   **Memory Complexity:** Requires a 3D matrix for quick access to adjacent voxels during encoding [S1, unofficial].
*   **Measured Performance:**
    *   Example voxels: 2.71 MB (2,842,706 bytes) in XYZI format vs 401 KB (410,931 bytes) in XYZI+CUBE format [S1, unofficial].
    *   Custom voxels (low color variety): 5.12 MB (5,377,820 bytes) in XYZI format vs 221 KB (226,475 bytes) in XYZI+CUBE format (1/24 reduction) [S1, unofficial].
*   **Scaling:** Drastically reduces file size for bigger models compared to XYZI [S1, unofficial].
*   **Strengths:** Decreases decoding time because an entire cube can be decoded at once [S1, unofficial].
*   **Limitations:** Encoding time is higher than raw XYZI. The "checkerbox problem" (alternating colors) is the worst-case scenario where no cubes are formed, resulting in storage equivalent to XYZI [S1, unofficial].
*   **When to use:** When models contain large solid blocks of uniform color and disk space is a constraint.
*   **When not to use:** When models consist primarily of alternating colors (checkerboards) or when encoding speed is prioritized over file size.

## Background
The XYZI+CUBE format was proposed by Foxiquek on March 4, 2022, in Issue #52 of the `ephtracy/voxel-model` repository [S1, unofficial]. It was implemented and detailed by JelleBouma starting August 8, 2022 [S1, unofficial]. The format is designed to be backward compatible with old encoders/decoders by making the CUBE chunk optional, though new files are not compatible with old decoders [S1, unofficial].

## Sources
[S1] Optional optimised format, https://github.com/ephtracy/voxel-model/issues/52, fetched 2026-10-28 (unofficial)
