# Tile-based Volume Ray Marching

Also called: Tiled Volume Ray Marching, 1D Texture Array Volume Rendering

## Overview
Tile-based volume ray marching is a dense grid rendering technique that stores 3D volume data in a 1D texture array organized into tiles, using ray marching to sample the volume and composite the result using front-to-back accumulation [S1, unofficial]. It solves the problem of mapping large 3D scalar fields onto 2D texture hardware by decomposing the Z-axis into discrete tiles, allowing efficient sampling via linear interpolation between adjacent tile slices [S1, unofficial]. It is distinguished from standard 3D texture sampling by its explicit tile-indexing arithmetic and from bricking by its reliance on a 1D texture array layout with linear interpolation rather than discrete brick selection [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D volume defined by normalized coordinates $(x, y, z) \in [0, 1]^3$.
*   **Input:** A 1D texture array `uVolData` storing the volume data, where each "slice" of the array corresponds to a 2D texture of size $uDimension \times uDimension$.
*   **Input:** Ray entry and exit coordinates provided as `frontColor` (entry point) and `backColor` (exit point) in screen space.
*   **Output:** Final accumulated color `gl_FragColor`.

**Data Structures**
The volume data is stored in a 1D texture array `uVolData`. The array is conceptually divided into $uTiles$ slices along the Z-axis. Each slice is a 2D texture of dimensions $uDimension \times uDimension$. The total number of texels in the 1D array is $uTiles \times uDimension \times uDimension$.

**Parameters**
*   $uTiles$: Number of tiles along the Z-axis [S1, unofficial].
*   $uTileWidth$: Width of each tile in pixels (typically equal to $uDimension$) [S1, unofficial].
*   $uTileHeight$: Height of each tile in pixels (typically equal to $uDimension$) [S1, unofficial].
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
    *   The entry point is stored in the varying `frontColor`.
    *   Compute the ray step vector:
        $$step = \frac{backColor.rgb - frontColor.rgb}{MAXSTEPS}$$
    *   Initialize the current position $vpos$ to `frontColor` and accumulators `accum` and `sample` to zero.

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
    *   Sample the volume at current position $vpos.xyz$ using `getTextureValue`. Let the result be $v$.
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
    *   **Termination**: Break if $vpos.x > 1.0$, $vpos.y > 1.0$, $vpos.z > 1.0$, or $accum.a \ge 1.0$.

4.  **Output**:
    *   Set `gl_FragColor = accum`.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tile_based_volume_ray_march(front_color, back_color, uTiles, uTileWidth, uTileHeight, uDimension, uMinimum, uRange, uOpacity, uLight, MAXSTEPS, get_texture_value):
    """
    Renders a volume using tile-based ray marching.
    
    Args:
        front_color: vec4, entry point of the ray (xyz) and initial color (rgba).
        back_color: vec4, exit point of the ray (xyz) and background color (rgba).
        uTiles: float, number of tiles along Z.
        uTileWidth: float, width of tile in pixels.
        uTileHeight: float, height of tile in pixels.
        uDimension: float, dimension of 2D texture sub-images.
        uMinimum: float, min volume value.
        uRange: float, volume value range (max - min).
        uOpacity: float, global opacity multiplier.
        uLight: float, global light multiplier.
        MAXSTEPS: int, max steps.
        get_texture_value: function, samples the tiled volume data.
        
    Returns:
        vec4, final accumulated color.
    """
    # Initialize accumulators
    accum = vec4(0.0, 0.0, 0.0, 0.0)
    sample = vec4(0.0, 0.0, 0.0, 0.0)
    
    # Ray setup
    # Note: In the source, front_color contains the ray origin, back_color contains the ray end.
    # The source code uses frontColor as the initial vpos.
    vpos = front_color
    
    # Compute step vector
    # The source code computes step as (backColor.rgb - frontColor.rgb) / MAXSTEPS
    # This assumes front_color.rgb is the ray origin and back_color.rgb is the ray destination.
    step = (back_color.rgb - vpos.rgb) / MAXSTEPS
    
    for i in range(MAXSTEPS):
        # Sample volume
        v = get_texture_value(vpos.xyz)
        
        # Compute sample properties
        # Source: sample.a = value.a * uOpacity * (1.0 / MAXSTEPS)
        # Source: sample.rgb = value.rgb * sample.a * uLight
        # Note: 'value' in source is the normalized scalar v.
        sample_a = v * uOpacity * (1.0 / MAXSTEPS)
        sample_rgb = v * sample_a * uLight
        
        # Accumulate
        accum.rgb += (1.0 - accum.a) * sample_rgb
        accum.a += sample_a
        
        # Advance
        vpos.xyz += step
        
        # Termination checks
        if vpos.x > 1.0 or vpos.y > 1.0 or vpos.z > 1.0 or accum.a >= 1.0:
            break
            
    return accum

def get_texture_value(volume_position, uTiles, uTileWidth, uTileHeight, uDimension, uMinimum, uRange, texture_sampler):
    """
    Samples the tiled volume data.
    
    Args:
        volume_position: vec3, normalized (0-1) coordinates.
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
    z = volume_position.z
    x = volume_position.x
    y = volume_position.y
    
    # Z-tile indices
    zp1 = floor(z * uTiles)
    zp2 = zp1 + 1.0
    
    # X/Y pixel coordinates
    xp = floor(x * uTileWidth)
    yp = floor(y * uTileHeight)
    
    # 1D array indices
    index1 = zp1 * uTileWidth * uTileHeight + yp * uTileWidth + xp
    index2 = zp2 * uTileWidth * uTileHeight + yp * uTileWidth + xp
    
    # 2D texture coordinates
    xt1 = mod(index1, uDimension)
    yt1 = floor(index1 / uDimension)
    
    xt2 = mod(index2, uDimension)
    yt2 = floor(index2 / uDimension)
    
    # Normalized coordinates
    tex_pos1 = vec2(xt1 / uDimension, yt1 / uDimension)
    tex_pos2 = vec2(xt2 / uDimension, yt2 / uDimension)
    
    # Sample
    val1 = texture_sampler(tex_pos1)
    val2 = texture_sampler(tex_pos2)
    
    # Normalize
    norm1 = (val1 - uMinimum) / uRange
    norm2 = (val2 - uMinimum) / uRange
    
    # Interpolate
    t = (z * uTiles) - zp1
    return mix(norm1, norm2, t)
```

**Edge Cases and Pitfalls**
*   **Boundary Conditions**: The loop breaks if $vpos > 1.0$. If the ray exits the volume bounds before $MAXSTEPS$, the loop terminates early, potentially leaving `accum` incomplete if the background is not handled separately (though the source uses `backColor` to define the ray end, implying the ray is always within the $[0,1]^3$ box for the marching steps).
*   **Precision**: The use of `floor` and `mod` for index calculation requires sufficient floating-point precision to avoid index collisions or gaps, especially for large $uDimension$ values.
*   **Normalization**: If $uRange$ is zero, division by zero occurs. This must be guarded against in implementation [G].
*   **Tile Alignment**: The method assumes $uTileWidth$ and $uTileHeight$ align with the texture dimensions. Mismatches can cause sampling errors at tile boundaries.

## Subtopics

## Operating characteristics
*   **Time Complexity**: $O(MAXSTEPS)$ per pixel, where $MAXSTEPS$ is the number of ray marching steps. The volume sampling function `getTextureValue` involves constant-time arithmetic and two texture lookups.
*   **Memory Complexity**: $O(uTiles \cdot uDimension^2)$ for the volume data storage. The method requires storing the volume in a 1D texture array.
*   **Performance**: Performance is limited by the number of ray steps and the cost of texture sampling. The method is efficient for moderate-resolution volumes but may suffer from texture cache thrashing if $uDimension$ is large or if the tile layout causes non-coherent memory access patterns.
*   **Strengths**: Simple implementation; leverages standard 2D texture hardware; linear interpolation provides smooth results across tile boundaries.
*   **Limitations**: Limited to dense grids; no adaptive step size; memory usage scales linearly with volume size; interpolation across tile boundaries may introduce artifacts if the volume data is not smooth or if the tile size is too small.
*   **When to use**: Suitable for real-time rendering of moderate-sized volumes on hardware with limited texture memory, where a simple, uniform sampling strategy is acceptable.
*   **When not to use**: Unsuitable for very large volumes requiring sparse representation, or for high-quality rendering requiring adaptive sampling or complex transfer functions that cannot be pre-computed into the texture.

## Background
The method is described in the `volumetric.js` library by kapadia [S1, unofficial]. It represents a practical approach to volume rendering in WebGL, where 3D textures are not always supported or efficient. By decomposing the volume into 1D texture array slices, it allows for efficient sampling using standard 2D texture units. The technique relies on front-to-back accumulation, a standard volume rendering algorithm, but adapts the sampling strategy to the constraints of the 1D texture array layout.

## Sources
[S1] kapadia/volumetric.js, https://github.com/kapadia/volumetric.js, fetched 2026-10-27 (unofficial)
