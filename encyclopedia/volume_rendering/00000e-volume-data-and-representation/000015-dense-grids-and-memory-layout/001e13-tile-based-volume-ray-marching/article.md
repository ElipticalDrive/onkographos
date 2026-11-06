# Tile-based Volume Ray Marching


## Overview
Tile-based volume ray marching is a dense grid rendering technique that stores 3D volume data in a 1D texture array organized into tiles, using ray marching to sample the volume and composite the result using front-to-back accumulation [S1, unofficial]. It solves the problem of mapping large 3D scalar fields onto 2D texture hardware by decomposing the Z-axis into discrete tiles, allowing efficient sampling via linear interpolation between adjacent tile slices [S1, unofficial]. It is distinguished from standard 3D texture sampling by its explicit tile-indexing arithmetic and from bricking by its reliance on a 1D texture array layout with linear interpolation rather than discrete brick selection [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D volume defined by normalized coordinates $(x, y, z) \in [0, 1]^3$ [S1, unofficial].
*   **Input:** Volume data stored in a 1D texture array `uVolData`, where each slice corresponds to a 2D texture of size $uDimension \times uDimension$ [S1, unofficial].
*   **Input:** Ray entry and exit coordinates provided as `frontColor` (entry point) and `backColor` (exit point) in screen space [S1, unofficial].
*   **Output:** Final accumulated color `gl_FragColor` [S1, unofficial].

**Data Structures**
The volume data is stored in a 1D texture array `uVolData` [S1, unofficial]. The array is conceptually divided into $uTiles$ slices along the Z-axis [S1, unofficial]. Each slice is a 2D texture of dimensions $uDimension \times uDimension$ [S1, unofficial]. The total number of texels in the 1D array is $uTiles \times uDimension \times uDimension$ [S1, unofficial].

**Parameters**
*   $uTiles$: Number of tiles along the Z-axis [S1, unofficial].
*   $uTileWidth$: Width of each tile in pixels [S1, unofficial].
*   $uTileHeight$: Height of each tile in pixels [S1, unofficial].
*   $uDimension$: Dimension (width/height) of the 2D texture sub-images [S1, unofficial].
*   $uMinimum$: Minimum scalar value in the volume [S1, unofficial].
*   $uMaximum$: Maximum scalar value in the volume [S1, unofficial].
*   $uRange$: Normalization range, $uMaximum - uMinimum$ [S1, unofficial].
*   $uOpacity$: Global opacity multiplier [S1, unofficial].
*   $uLight$: Global lighting intensity multiplier [S1, unofficial].
*   $MAXSTEPS$: Constant integer representing the maximum number of ray marching steps [S1, unofficial].

**Algorithm Steps**

1.  **Ray Initialization**:
    *   Compute normalized screen texture coordinates from the varying `position`:
        $$textureCoord = \frac{position.xy}{position.w}$$
        $$textureCoord.x = 0.5 \cdot textureCoord.x + 0.5$$
        $$textureCoord.y = 0.5 \cdot textureCoord.y + 0.5$$
    *   Fetch the back color (exit point) from `uBackCoord` at `textureCoord`:
        $$backColor = \text{texture2D}(uBackCoord, textureCoord)$$
    *   The entry point is stored in the varying `frontColor` [S1, unofficial].
    *   Compute the ray step vector:
        $$step = \frac{backColor.rgb - frontColor.rgb}{MAXSTEPS}$$
    *   Initialize the current position $vpos$ to `frontColor` and accumulators `accum` and `sample` to zero [S1, unofficial].

2.  **Volume Sampling (`getTextureValue`)**:
    For a given volume position $P = (x, y, z)$:
    *   Calculate the Z-tile indices for the current position and the next slice:
        $$zp1 = \lfloor z \cdot uTiles \rfloor$$
        $$zp2 = zp1 + 1$$
    *   Calculate the X and Y pixel coordinates within the tile:
        $$xp = \lfloor x \cdot uTileWidth \rfloor$$
        $$yp = \lfloor y \cdot uTileHeight \rfloor$$
    *   Calculate the 1D array indices for the two Z-slices:
        $$index1 = zp1 \cdot uTileWidth \cdot uTileHeight + yp \cdot uTileWidth + xp$$
        $$index2 = zp2 \cdot uTileWidth \cdot uTileHeight + yp \cdot uTileWidth + xp$$
    *   Convert 1D indices to 2D texture coordinates $(xt, yt)$ for the 2D sub-textures:
        $$xt1 = \text{mod}(index1, uDimension)$$
        $$yt1 = \lfloor index1 / uDimension \rfloor$$
        $$xt2 = \text{mod}(index2, uDimension)$$
        $$yt2 = \lfloor index2 / uDimension \rfloor$$
    *   Normalize to $[0, 1]$ range:
        $$texPos1 = \left( \frac{xt1}{uDimension}, \frac{yt1}{uDimension} \right)$$
        $$texPos2 = \left( \frac{xt2}{uDimension}, \frac{yt2}{uDimension} \right)$$
    *   Sample the volume data:
        $$val1 = \text{texture2D}(uVolData, texPos1).x$$
        $$val2 = \text{texture2D}(uVolData, texPos2).x$$
    *   Normalize values to $[0, 1]$:
        $$norm1 = \frac{val1 - uMinimum}{uRange}$$
        $$norm2 = \frac{val2 - uMinimum}{uRange}$$
    *   Interpolate based on the fractional Z position within the tile:
        $$t = (z \cdot uTiles) - zp1$$
        $$\text{return mix}(norm1, norm2, t)$$

3.  **Ray Marching Loop**:
    Iterate $i$ from $0$ to $MAXSTEPS - 1$:
    *   Sample the volume at current position $vpos.xyz$ using `getTextureValue`. Let the result be $v$ [S1, unofficial].
    *   Set transfer function input:
        $$transferPosition.x = v$$
        $$transferPosition.y = 0.5$$
    *   Compute sample alpha and RGB:
        $$sample.a = v \cdot uOpacity \cdot \frac{1.0}{MAXSTEPS}$$
        $$sample.rgb = v \cdot sample.a \cdot uLight$$
    *   Front-to-back accumulation:
        $$accum.rgb += (1.0 - accum.a) \cdot sample.rgb$$
        $$accum.a += sample.a$$
    *   Advance position:
        $$vpos.xyz += step$$
    *   **Termination**: Break if $vpos.x > 1.0$, $vpos.y > 1.0$, $vpos.z > 1.0$, or $accum.a \ge 1.0$ [S1, unofficial].

4.  **Output**:
    *   Set `gl_FragColor = accum` [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tile_based_volume_ray_march(front_color, back_color, uTiles, uTileWidth, uTileHeight, uDimension, uMinimum, uRange, uOpacity, uLight, MAXSTEPS, texture_sampler):
    """
    Renders a volume using tile-based ray marching.
    
    Args:
        front_color: tuple/list of 4 floats, ray origin (xyz) and initial color (rgba).
        back_color: tuple/list of 4 floats, ray exit point (xyz) and background color (rgba).
        uTiles: float, number of tiles along Z.
        uTileWidth: float, width of tile in pixels.
        uTileHeight: float, height of tile in pixels.
        uDimension: float, dimension of 2D texture sub-images.
        uMinimum: float, min volume value.
        uRange: float, volume value range (max - min).
        uOpacity: float, global opacity multiplier.
        uLight: float, global light multiplier.
        MAXSTEPS: int, max steps.
        texture_sampler: function, samples a 2D texture at normalized (u, v) coordinates.
        
    Returns:
        list of 4 floats, final accumulated color [r, g, b, a].
    """
    # Initialize accumulators
    accum = [0.0, 0.0, 0.0, 0.0]
    sample = [0.0, 0.0, 0.0, 0.0]
    
    # Ray setup: vpos starts at front_color.xyz
    vpos = [front_color[0], front_color[1], front_color[2]]
    
    # Compute step vector
    step = [(back_color[0] - front_color[0]) / MAXSTEPS, 
            (back_color[1] - front_color[1]) / MAXSTEPS, 
            (back_color[2] - front_color[2]) / MAXSTEPS]
    
    for _ in range(MAXSTEPS):
        # Sample volume
        v = get_texture_value(vpos, uTiles, uTileWidth, uTileHeight, uDimension, uMinimum, uRange, texture_sampler)
        
        # Compute sample properties
        sample_a = v * uOpacity * (1.0 / MAXSTEPS)
        # Source scales all RGB channels equally by the scalar value
        sample_rgb = [v * sample_a * uLight, v * sample_a * uLight, v * sample_a * uLight]
        
        # Accumulate
        accum[0] += (1.0 - accum[3]) * sample_rgb[0]
        accum[1] += (1.0 - accum[3]) * sample_rgb[1]
        accum[2] += (1.0 - accum[3]) * sample_rgb[2]
        accum[3] += sample_a
        
        # Advance
        vpos = [vpos[0] + step[0], vpos[1] + step[1], vpos[2] + step[2]]
        
        # Termination checks
        if vpos[0] > 1.0 or vpos[1] > 1.0 or vpos[2] > 1.0 or accum[3] >= 1.0:
            break
            
    return accum

def get_texture_value(volume_position, uTiles, uTileWidth, uTileHeight, uDimension, uMinimum, uRange, texture_sampler):
    """
    Samples the tiled volume data.
    
    Args:
        volume_position: tuple/list of 3 floats, normalized (0-1) coordinates.
        uTiles: float.
        uTileWidth: float.
        uTileHeight: float.
        uDimension: float.
        uMinimum: float.
        uRange: float.
        texture_sampler: function, samples a 2D texture at normalized coordinates.
        
    Returns:
        float, normalized volume value.
    """
    z, x, y = volume_position
    
    # Z-tile indices
    zp1 = int(z * uTiles)  # floor
    zp2 = zp1 + 1
    
    # X/Y pixel coordinates
    xp = int(x * uTileWidth)
    yp = int(y * uTileHeight)
    
    # 1D array indices
    index1 = zp1 * uTileWidth * uTileHeight + yp * uTileWidth + xp
    index2 = zp2 * uTileWidth * uTileHeight + yp * uTileWidth + xp
    
    # 2D texture coordinates
    xt1 = index1 % uDimension
    yt1 = index1 // uDimension
    
    xt2 = index2 % uDimension
    yt2 = index2 // uDimension
    
    # Normalized coordinates
    tex_pos1 = (xt1 / uDimension, yt1 / uDimension)
    tex_pos2 = (xt2 / uDimension, yt2 / uDimension)
    
    # Sample
    val1 = texture_sampler(tex_pos1)
    val2 = texture_sampler(tex_pos2)
    
    # Normalize
    if uRange == 0:
        return 0.0  # Guard against division by zero [G]
    norm1 = (val1 - uMinimum) / uRange
    norm2 = (val2 - uMinimum) / uRange
    
    # Interpolate
    t = (z * uTiles) - zp1
    return norm1 + t * (norm2 - norm1)
```

**Edge Cases and Pitfalls**
*   **Boundary Conditions**: The loop breaks if $vpos > 1.0$ after advancing. If the ray origin is outside the $[0,1]^3$ box, the loop terminates immediately, returning zero accumulation [S1, unofficial].
*   **Precision**: The use of `floor` and `mod` for index calculation requires sufficient floating-point precision to avoid index collisions or gaps, especially for large $uDimension$ values [G].
*   **Normalization**: If $uRange$ is zero, division by zero occurs. This must be guarded against in implementation [G].
*   **Tile Alignment**: The method assumes $uTileWidth$ and $uTileHeight$ align with the texture dimensions. Mismatches can cause sampling errors at tile boundaries [G].

## Subtopics

## Operating characteristics
*   **Time Complexity**: $O(MAXSTEPS)$ per pixel, where $MAXSTEPS$ is the number of ray marching steps [S1, unofficial]. The volume sampling function involves constant-time arithmetic and two texture lookups [S1, unofficial].
*   **Memory Complexity**: $O(uTiles \cdot uDimension^2)$ for the volume data storage [S1, unofficial].
*   **Performance**: The method is efficient for moderate-resolution volumes but may suffer from texture cache thrashing if $uDimension$ is large or if the tile layout causes non-coherent memory access patterns [G].
*   **Strengths**: Simple implementation; leverages standard 2D texture hardware; linear interpolation provides smooth results across tile boundaries [S1, unofficial].
*   **Limitations**: Limited to dense grids; no adaptive step size; memory usage scales linearly with volume size; interpolation across tile boundaries may introduce artifacts if the volume data is not smooth or if the tile size is too small [G].
*   **When to use**: Suitable for real-time rendering of moderate-sized volumes on hardware with limited texture memory, where a simple, uniform sampling strategy is acceptable [G].
*   **When not to use**: Unsuitable for very large volumes requiring sparse representation, or for high-quality rendering requiring adaptive sampling or complex transfer functions that cannot be pre-computed into the texture [G].

## Background
The method is a practical approach to volume rendering in WebGL, where 3D textures are not always supported or efficient [S1, unofficial]. By decomposing the volume into 1D texture array slices, it allows for efficient sampling using standard 2D texture units [S1, unofficial]. The technique relies on front-to-back accumulation, a standard volume rendering algorithm, but adapts the sampling strategy to the constraints of the 1D texture array layout [S1, unofficial].

## Sources
[S1] kapadia/volumetric.js, https://github.com/kapadia/volumetric.js, fetched 2026-10-27 (unofficial)
