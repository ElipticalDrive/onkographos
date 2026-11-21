# Tile-based Volume Ray Marching

Also called: Tiled Volume Ray Marching, Z-Tiled Texture Array Raycasting

## Overview
Tile-based Volume Ray Marching is a GPU-accelerated volume rendering technique that stores 3D scalar field data in a 1D texture array organized into 2D tiles, using front-to-back ray marching to sample the volume and composite the final image [S1, unofficial]. It solves the memory bandwidth and caching inefficiencies of monolithic 3D textures by decomposing the volume domain into smaller, manageable 2D sub-images, allowing the renderer to access data in cache-friendly 2D blocks while maintaining continuous interpolation across tile boundaries [S1, unofficial]. It sets itself apart from [Brick-based Volume Raycasting](3341-brick-based-volume-raycasting.md) by using a fixed, uniform tiling structure mapped directly to texture coordinates without complex brick metadata or hierarchical traversal, and from [Brick Caching with LRU Replacement](879-brick-caching-with-lru-replacement.md) by relying on hardware texture filtering for interpolation rather than software-managed cache eviction policies.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset stored as a 1D texture array composed of 2D tiles. Uniform parameters defining the tiling geometry (`uTiles`, `uTileWidth`, `uTileHeight`, `uDimension`), data normalization (`uMinimum`, `uMaximum`, `uRange`), rendering properties (`uOpacity`, `uLight`), and step count (`MAXSTEPS`). A `uBackCoord` texture providing the back-plane intersection coordinates for each pixel.
*   **Output:** A 2D color buffer (`gl_FragColor`) containing the composited volume image.

**Data Structures:**
*   **Volume Texture (`uVolData`):** A 2D texture array where each layer corresponds to a 2D tile. The total number of layers is determined by the Z-axis tiling. Each tile has dimensions `uTileWidth` x `uTileHeight`. The internal storage dimension is `uDimension` (typically a power of two).
*   **Back Coordination Texture (`uBackCoord`):** A 2D texture storing the 3D world coordinates of the ray's entry and exit points (or back-plane intersection) for each screen pixel.

**Algorithm Steps:**

1.  **Ray Initialization:**
    For each pixel, retrieve the back-plane intersection color/position from `uBackCoord`. Let `frontColor` be the varying position passed from the vertex shader (representing the ray origin or front-plane intersection) and `backColor` be the sampled back-plane position.
    Calculate the ray direction vector `step` as:
    $$ \text{step} = \frac{\text{backColor.rgb} - \text{frontColor.rgb}}{\text{MAXSTEPS}} $$
    Initialize the current ray position `vpos` to `frontColor`. Initialize the accumulated color `accum` and sample `sample` to zero vectors.

2.  **Ray Marching Loop:**
    Iterate `i` from 0 to `MAXSTEPS - 1`:
    
    a. **Sample Volume Data:**
       Call the `getTextureValue` function with the current ray position `vpos.xyz` to retrieve the normalized scalar value at the current step.
       
       *Implementation of `getTextureValue(vpos)`:*
       1.  Calculate the Z-tile index for the current position:
           $$ z_{p1} = \lfloor \text{vpos.z} \times \text{uTiles} \rfloor $$
       2.  Calculate the Z-tile index for the next slice:
           $$ z_{p2} = z_{p1} + 1.0 $$
       3.  Calculate the X and Y pixel coordinates within the tile:
           $$ x_p = \lfloor \text{uTileWidth} \times \text{vpos.x} \rfloor $$
           $$ y_p = \lfloor \text{uTileHeight} \times \text{vpos.y} \rfloor $$
       4.  Calculate the 1D array indices for the two adjacent Z-tiles:
           $$ \text{index}_1 = z_{p1} \times \text{uTileWidth} \times \text{uTileHeight} + y_p \times \text{uTileWidth} + x_p $$
           $$ \text{index}_2 = z_{p2} \times \text{uTileWidth} \times \text{uTileHeight} + y_p \times \text{uTileWidth} + x_p $$
       5.  Convert 1D indices to 2D texture coordinates within the `uDimension` x `uDimension` sub-images:
           $$ xt_1 = \text{mod}(\text{index}_1, \text{uDimension}) $$
           $$ yt_1 = \lfloor \text{index}_1 / \text{uDimension} \rfloor $$
           $$ xt_2 = \text{mod}(\text{index}_2, \text{uDimension}) $$
           $$ yt_2 = \lfloor \text{index}_2 / \text{uDimension} \rfloor $$
       6.  Normalize these coordinates to the [0, 1] range:
           $$ \text{texPos}_1 = \left( \frac{xt_1}{\text{uDimension}}, \frac{yt_1}{\text{uDimension}} \right) $$
           $$ \text{texPos}_2 = \left( \frac{xt_2}{\text{uDimension}}, \frac{yt_2}{\text{uDimension}} \right) $$
       7.  Sample the raw scalar values from the volume texture:
           $$ v_1 = \text{texture2D}(\text{uVolData}, \text{texPos}_1).x $$
           $$ v_2 = \text{texture2D}(\text{uVolData}, \text{texPos}_2).x $$
       8.  Normalize the values to the [0, 1] range using the global min/max:
           $$ v_1 = \frac{v_1 - \text{uMinimum}}{\text{uRange}} $$
           $$ v_2 = \frac{v_2 - \text{uMinimum}}{\text{uRange}} $$
       9.  Interpolate between the two Z-tile values based on the fractional Z position within the current tile:
           $$ \text{fraction} = (\text{vpos.z} \times \text{uTiles}) - z_{p1} $$
           $$ \text{return} \, \text{mix}(v_1, v_2, \text{fraction}) $$

    b. **Transfer Function and Lighting:**
       Set `transferPosition.x` to the sampled value and `transferPosition.y` to 0.5 (placeholder for color mapping, though the source implementation uses the scalar value directly as RGBA).
       $$ \text{value} = \text{vec4}(\text{transferPosition.x}) $$
       Compute sample alpha:
       $$ \text{sample.a} = \text{value.a} \times \text{uOpacity} \times \frac{1.0}{\text{MAXSTEPS}} $$
       Compute sample RGB (assuming `value.rgb` is derived from the scalar or a lookup, here simplified to scalar multiplication as per source):
       $$ \text{sample.rgb} = \text{value.rgb} \times \text{sample.a} \times \text{uLight} $$

    c. **Front-to-Back Accumulation:**
       Update the accumulated color and alpha:
       $$ \text{accum.rgb} += (1.0 - \text{accum.a}) \times \text{sample.rgb} $$
       $$ \text{accum.a} += \text{sample.a} $$

    d. **Ray Advancement and Termination:**
       Advance the ray position:
       $$ \text{vpos.xyz} += \text{step} $$
       Break the loop if `vpos.x > 1.0`, `vpos.y > 1.0`, `vpos.z > 1.0` (ray exited volume bounds), or `accum.a >= 1.0` (ray is fully opaque).

3.  **Output:**
    Set `gl_FragColor` to `accum`.

**Parameters:**
*   `uTiles`: Number of tiles along the Z-axis.
*   `uTileWidth`, `uTileHeight`: Dimensions of each 2D tile in pixels.
*   `uDimension`: The internal dimension (width/height) of the 2D texture sub-images, typically a power of two.
*   `uMinimum`, `uMaximum`: Min and max scalar values in the volume.
*   `uRange`: `uMaximum - uMinimum`.
*   `uOpacity`: Global opacity multiplier.
*   `uLight`: Global lighting intensity multiplier.
*   `MAXSTEPS`: Constant integer defining the number of steps per ray.

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The loop termination checks `vpos > 1.0`. If the ray origin is outside [0,1] or the step vector is not normalized to the volume bounds, the ray may march indefinitely or sample invalid memory. The source assumes `frontColor` and `backColor` are correctly clamped or mapped to [0,1].
*   **Precision:** The use of `floor` and `mod` for texture coordinate calculation can introduce precision artifacts at tile boundaries if `uDimension` is large or floating-point precision is low. [G] Ensure `highp` precision is used in the shader.
*   **Normalization:** If `uRange` is zero, division by zero occurs. A guard should check for this.

## Subtopics

### [Tiled Fitting with Hann Tapering](3103-tiled-fitting-with-hann-tapering.md)
A volume data fitting strategy that partitions large 3D volumes into overlapping tiles to fit Gaussian splat models independently. It distinguishes itself by applying a Hann taper in overlap regions to ensure a partition-of-unity and prevent visible seams. Choose this when rendering large volumes on hardware with limited memory and Gaussian splatting is the target representation.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is `MAXSTEPS`. The per-step cost is constant, involving a few arithmetic operations and two texture lookups.
*   **Memory Complexity:** $O(V)$, where $V$ is the total volume of data. The data is stored in a 1D texture array. The memory footprint is determined by the total number of voxels and the texture format.
*   **Performance:** Performance is heavily dependent on texture cache efficiency. By organizing data into 2D tiles, spatial locality is improved for rays moving primarily in the XY plane, but rays moving in Z may suffer from cache misses if tiles are not aligned with the ray direction. [S1, unofficial]
*   **Strengths:** Simple implementation; leverages standard 2D texture hardware filtering; no complex data structures or traversal logic required.
*   **Limitations:** Fixed tiling structure may not adapt to data density; interpolation across tile boundaries relies on texture filtering which may not be perfect for all transfer functions; no adaptive step size.
*   **When to use:** Small to medium volumes where simplicity is preferred; hardware that supports efficient 2D texture arrays.
*   **When not to use:** Very large volumes requiring hierarchical culling or adaptive sampling; scenes with highly anisotropic data where fixed tiles cause severe cache misses.

## Background
Tile-based Volume Ray Marching was implemented in the `volumetric.js` library by kapadia [S1, unofficial]. It represents a practical approach to GPU volume rendering using WebGL, focusing on simplicity and compatibility with standard texture units. The method builds on the principles of brick-based storage but simplifies the access pattern by using a uniform tile grid mapped directly to texture coordinates.

## Sources
[S1] kapadia/volumetric.js, https://github.com/kapadia/volumetric.js, fetched 2026-10-27 (unofficial)
