# Tile-based Volume Ray Marching

Also called: Tiled Volume Ray Marching, Z-tiled Volume Rendering

## Overview
Tile-based Volume Ray Marching is a GPU-accelerated volume rendering technique that stores 3D scalar field data in a 1D texture array organized into 2D tiles, using front-to-back ray marching to sample and composite the volume [S1, unofficial]. It solves the problem of managing large volume datasets within limited GPU texture memory by decomposing the Z-axis into discrete tiles, allowing the renderer to access only the relevant tile slices for a given ray segment [S1, unofficial]. It is distinguished from standard 3D texture mapping by its explicit 1D-to-2D coordinate conversion logic and linear interpolation between tile boundaries, and from brick-based storage by its simpler, non-hierarchical tiling structure [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume defined by normalized coordinates $(x, y, z) \in [0, 1]^3$. The data is stored in a 2D texture `uVolData` of dimensions $D \times D$ (where $D = uDimension$), which acts as a flattened 1D array of tiles. Each tile has dimensions $uTileWidth \times uTileHeight$. The number of tiles along the Z-axis is $uTiles$.
*   **Uniforms:** `uMinimum`, `uMaximum`, `uRange` (for normalization), `uOpacity`, `uLight`, `MAXSTEPS` (constant integer), `uTiles`, `uTileWidth`, `uTileHeight`, `uDimension`.
*   **Output:** A final RGBA color `gl_FragColor` representing the accumulated volume along the ray.

**Data Structures:**
The volume data is stored in a single 2D texture `uVolData`. The logical layout is a 1D array of tiles, where each tile is a 2D image. The total number of elements in the 1D array is $N = uTiles \times uTileWidth \times uTileHeight$. This 1D index is mapped to 2D texture coordinates $(xt, yt)$ using modulo and division by $uDimension$.

**Algorithm Steps:**

1.  **Ray Initialization:**
    *   Compute normalized texture coordinates from the fragment position: $coord = \frac{position.xy}{position.w}$, then map to $[0, 1]$: $coord = 0.5 \cdot coord + 0.5$.
    *   Fetch the back color from `uBackCoord` texture: $C_{back} = \text{texture2D}(uBackCoord, coord)$.
    *   Define the ray origin $vpos$ as the front color (passed as a varying `frontColor`).
    *   Compute the step vector: $\vec{s} = \frac{C_{back}.rgb - vpos.rgb}{MAXSTEPS}$.
    *   Initialize accumulation buffer: $A = (0, 0, 0, 0)$.

2.  **Ray Marching Loop:**
    *   Iterate $i$ from $0$ to $MAXSTEPS - 1$:
        *   **Sample Volume:** Call `getTextureValue(vpos.xyz)` to retrieve the normalized scalar value $v$ at the current ray position.
        *   **Transfer Function:** Map $v$ to RGBA. The source uses a simplified transfer function where the alpha is derived from the scalar value and RGB is set to the scalar value (or a constant). Specifically: $T_{pos} = (v, 0.5)$. The sample value is $S_{val} = (v, v, v, v)$ (assuming RGB equals the scalar for visualization, or as defined by the specific transfer function implementation).
        *   **Compute Sample Properties:**
            *   Sample Alpha: $\alpha_s = S_{val}.a \cdot uOpacity \cdot \frac{1.0}{MAXSTEPS}$.
            *   Sample RGB: $\vec{c}_s = S_{val}.rgb \cdot \alpha_s \cdot uLight$.
        *   **Front-to-Back Accumulation:**
            *   $\vec{A}.rgb += (1.0 - A.a) \cdot \vec{c}_s$.
            *   $A.a += \alpha_s$.
        *   **Advance Ray:** $vpos.xyz += \vec{s}$.
        *   **Termination Check:** Break if $vpos.x > 1.0$ or $vpos.y > 1.0$ or $vpos.z > 1.0$ or $A.a \ge 1.0$.

3.  **Output:**
    *   Set `gl_FragColor = A`.

**`getTextureValue` Function Details:**
This function handles the mapping from 3D volume coordinates to the tiled 2D texture.

1.  **Tile Indexing:**
    *   Calculate the Z-tile index for the current position: $zp1 = \lfloor vpos.z \cdot uTiles \rfloor$.
    *   Calculate the Z-tile index for the next slice: $zp2 = zp1 + 1.0$.
    *   Calculate the X-pixel coordinate within the tile: $xp = \lfloor vpos.x \cdot uTileWidth \rfloor$.
    *   Calculate the Y-pixel coordinate within the tile: $yp = \lfloor vpos.y \cdot uTileHeight \rfloor$.

2.  **1D Array Indexing:**
    *   Calculate the 1D index for the first tile: $idx1 = zp1 \cdot uTileWidth \cdot uTileHeight + yp \cdot uTileWidth + xp$.
    *   Calculate the 1D index for the second tile: $idx2 = zp2 \cdot uTileWidth \cdot uTileHeight + yp \cdot uTileWidth + xp$.

3.  **2D Texture Coordinate Conversion:**
    *   Convert $idx1$ to 2D coordinates: $xt1 = idx1 \pmod{uDimension}$, $yt1 = \lfloor idx1 / uDimension \rfloor$.
    *   Convert $idx2$ to 2D coordinates: $xt2 = idx2 \pmod{uDimension}$, $yt2 = \lfloor idx2 / uDimension \rfloor$.

4.  **Normalization:**
    *   Normalize for first sample: $texPos1 = (\frac{xt1}{uDimension}, \frac{yt1}{uDimension})$.
    *   Normalize for second sample: $texPos2 = (\frac{xt2}{uDimension}, \frac{yt2}{uDimension})$.

5.  **Sampling and Interpolation:**
    *   Sample values: $val1 = \text{texture2D}(uVolData, texPos1).x$, $val2 = \text{texture2D}(uVolData, texPos2).x$.
    *   Normalize values: $val1 = \frac{val1 - uMinimum}{uRange}$, $val2 = \frac{val2 - uMinimum}{uRange}$.
    *   Interpolate based on fractional Z position within the tile: $t = (vpos.z \cdot uTiles) - zp1$.
    *   Return $\text{mix}(val1, val2, t)$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tile_based_volume_ray_marching(front_color, back_color, volume_data_texture, uTiles, uTileWidth, uTileHeight, uDimension, uMinimum, uRange, uOpacity, uLight, MAXSTEPS):
    """
    Simulates the GLSL ray marching loop for tile-based volume rendering.
    
    Args:
        front_color: vec4, ray origin (front surface color/position)
        back_color: vec4, ray end point (back surface color/position)
        volume_data_texture: function(x, y) -> float, samples the 2D texture at normalized coords
        uTiles: float, number of tiles along Z
        uTileWidth: float, width of each tile in pixels
        uTileHeight: float, height of each tile in pixels
        uDimension: float, dimension of the 2D texture sub-images
        uMinimum: float, minimum volume value
        uRange: float, range of volume values (max - min)
        uOpacity: float, global opacity multiplier
        uLight: float, global lighting multiplier
        MAXSTEPS: int, number of steps per ray
        
    Returns:
        vec4, accumulated color
    """
    
    # Ray initialization
    step = (back_color.rgb - front_color.rgb) / MAXSTEPS
    vpos = front_color.xyz
    accum = [0.0, 0.0, 0.0, 0.0] # rgba
    
    for i in range(MAXSTEPS):
        # Termination check
        if vpos[0] > 1.0 or vpos[1] > 1.0 or vpos[2] > 1.0 or accum[3] >= 1.0:
            break
            
        # Sample volume
        val = get_texture_value(vpos, uTiles, uTileWidth, uTileHeight, uDimension, uMinimum, uRange, volume_data_texture)
        
        # Transfer function (simplified from source: RGB=scalar, Alpha=scalar*opacity/STEPS)
        # Note: Source sets value.a = value.a * uOpacity * (1.0/MAXSTEPS)
        # And value.rgb = value.rgb * sample.a * uLight
        # Assuming value.rgb is the scalar 'val' for this simplified model
        sample_alpha = val * uOpacity * (1.0 / MAXSTEPS)
        sample_rgb = [val, val, val] # Assuming RGB maps to scalar value
        
        # Accumulate
        weight = 1.0 - accum[3]
        accum[0] += weight * sample_rgb[0] * uLight
        accum[1] += weight * sample_rgb[1] * uLight
        accum[2] += weight * sample_rgb[2] * uLight
        accum[3] += sample_alpha
        
        # Advance
        vpos[0] += step[0]
        vpos[1] += step[1]
        vpos[2] += step[2]
        
    return accum

def get_texture_value(vpos, uTiles, uTileWidth, uTileHeight, uDimension, uMinimum, uRange, texture_sampler):
    """
    Maps 3D volume position to interpolated value in tiled 2D texture.
    """
    # Tile indices
    zp1 = floor(vpos[2] * uTiles)
    zp2 = zp1 + 1.0
    
    # Pixel coordinates within tile
    xp = floor(vpos[0] * uTileWidth)
    yp = floor(vpos[1] * uTileHeight)
    
    # 1D indices
    idx1 = zp1 * uTileWidth * uTileHeight + yp * uTileWidth + xp
    idx2 = zp2 * uTileWidth * uTileHeight + yp * uTileWidth + xp
    
    # 2D texture coordinates
    xt1 = idx1 % uDimension
    yt1 = floor(idx1 / uDimension)
    
    xt2 = idx2 % uDimension
    yt2 = floor(idx2 / uDimension)
    
    # Normalized coordinates
    tex_pos1 = (xt1 / uDimension, yt1 / uDimension)
    tex_pos2 = (xt2 / uDimension, yt2 / uDimension)
    
    # Sample
    val1 = texture_sampler(tex_pos1[0], tex_pos1[1])
    val2 = texture_sampler(tex_pos2[0], tex_pos2[1])
    
    # Normalize
    val1 = (val1 - uMinimum) / uRange
    val2 = (val2 - uMinimum) / uRange
    
    # Interpolate
    t = (vpos[2] * uTiles) - zp1
    return mix(val1, val2, t)

def mix(a, b, t):
    return a + t * (b - a)

def floor(x):
    return int(x) if x >= 0 else int(x) - (1 if x != int(x) else 0)

def mod(a, b):
    return a % b
```

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The loop breaks if $vpos > 1.0$. If the ray exits the volume early, remaining steps are skipped.
*   **Precision:** Using `floor` for coordinate mapping can introduce quantization errors at tile boundaries. The linear interpolation between $zp1$ and $zp2$ mitigates this for the Z-axis, but X and Y are sampled directly from the tile, potentially causing discontinuities if the tile boundaries do not align with the ray step size.
*   **Texture Sampling:** The source uses `texture2D` on a 2D texture that stores 1D array data. This requires careful management of `uDimension` to ensure correct wrapping and indexing. If `uDimension` is not a power of two, modulo arithmetic must be precise.
*   **Normalization:** If `uRange` is zero, division by zero occurs. A guard should ensure `uRange > 0`.

## Subtopics

### [Tiled Fitting with Hann Tapering](3103-tiled-fitting-with-hann-tapering.md)
A variant that applies Hann tapering to tile boundaries to eliminate seams in Gaussian splat models.

### [Brick-based storage and caching](4138-brick-based-storage-and-caching.md)
A hierarchical tiling approach that caches bricks in memory.

## Operating characteristics
*   **Time Complexity:** $O(MAXSTEPS)$ per pixel, where $MAXSTEPS$ is the number of ray marching steps. Each step involves a constant number of arithmetic operations and two texture lookups (for interpolation).
*   **Memory Complexity:** $O(uTiles \cdot uTileWidth \cdot uTileHeight)$ for the volume data texture. This is significantly less than a full 3D texture for large volumes, as it only stores the necessary tiles.
*   **Performance:** The method is efficient for real-time rendering on GPUs due to the use of 2D texture sampling and simple arithmetic. However, the lack of hierarchical culling (like in VDB or brick-based methods) means all steps along the ray are computed, even in empty space.
*   **Strengths:** Simple implementation, low memory footprint compared to full 3D textures, easy to integrate with existing 2D texture pipelines.
*   **Limitations:** No early ray termination based on empty space (only opacity-based), potential artifacts at tile boundaries if not interpolated correctly, limited to uniform tiling.
*   **When to Use:** Suitable for real-time volume rendering of moderately sized volumes where memory is constrained and hierarchical acceleration structures are not feasible.
*   **When Not to Use:** For very large volumes with significant empty space, where hierarchical methods like VDB or brick-based rendering would offer better performance through early termination.

## Background
Tile-based Volume Ray Marching was introduced in the `volumetric.js` library by kapadia [S1, unofficial]. It is designed for web-based volume rendering using WebGL, leveraging the texture array capabilities of the GPU to manage large datasets. The method builds on the principles of dense grid storage and linear indexing, adapting them for a tiled 2D texture layout.

## Sources
[S1] kapadia/volumetric.js, https://github.com/kapadia/volumetric.js, fetched 2026-10-27 (unofficial)
