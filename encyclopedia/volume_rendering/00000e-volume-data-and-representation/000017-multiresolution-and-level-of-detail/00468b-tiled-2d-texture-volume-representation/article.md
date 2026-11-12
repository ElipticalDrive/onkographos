# Tiled 2D Texture Volume Representation

Also called: 2D texture tiling for 3D data, Channel tiling

## Overview
Tiled 2D Texture Volume Representation is a technique for storing and sampling 3D volumetric data within a 2D texture by tiling the spatial dimensions (x, y) across the texture's width and height, and mapping the depth dimension (z) to the RGB color channels [S1, unofficial]. It solves the problem of rendering 3D volumes on hardware or APIs that lack native 3D texture support (e.g., early WebGL) by flattening the volume into a 2D format [S1, unofficial]. It differs from brick-based methods like [Window/Level Volume Raycasting with Bricking](2775-window-level-volume-rendering-with-bricking.md) by using fixed-size rectangular tiles arranged in a grid, and from channel-tiled methods that might use RGBA for different attributes, by strictly mapping z-slices to RGB channels, which limits the effective depth resolution to 3 slices per tile unless multiple textures or complex indexing are used [S1, unofficial].

## How it works
**Inputs:**
*   `dataTexture`: A 2D RGBA texture containing the tiled 3D volume data.
*   `firstPassTexture`: A 2D texture storing the back-face depth/position of the volume bounding box for ray casting.
*   `dataShape`: A `vec3` representing the dimensions of the original 3D volume $(N_x, N_y, N_z)$.
*   `textureShape`: A `vec2` representing the dimensions of the 2D texture tiles $(T_x, T_y)$.
*   `alphaCorrection`: A scalar uniform to convert raw data values to alpha.
*   `steps`: The number of ray marching steps.

**Outputs:**
*   Final composited color and alpha for each pixel.

**Data Structures:**
The 2D texture is organized into tiles. Each tile covers a sub-region of the $x$ and $y$ dimensions. The $z$ dimension is mapped to the RGB channels. Specifically, within a tile, the Red channel corresponds to $z$-slice $k$, Green to $k+1$, and Blue to $k+2$ [S1, unofficial]. The source code implies a specific tiling order: column-major for x, then row-major for y, then channel [S1, unofficial].

**Parameters:**
*   `MAX_STEPS`: 512 [S1, unofficial].
*   `dataShape`: Example `vec3(99, 93, 330)` [S1, unofficial].
*   `textureShape`: Example `vec2(1024, 1024)` [S1, unofficial].

**Algorithm:**

1.  **Calculate Tile Grid:**
    Determine the number of tiles in x and y:
    $$ nTiles.x = \lfloor T_x / N_x \rfloor $$
    $$ nTiles.y = \lfloor T_y / N_y \rfloor $$
    Calculate tiles per layer (z-slice group):
    $$ tilesPerLayer = nTiles.x \times nTiles.y $$
    Calculate tile dimensions in texture space (normalized 0-1):
    $$ tileDim.x = 1.0 / (T_x / N_x) $$
    $$ tileDim.y = 1.0 / (T_y / N_y) $$

2.  **Determine Z-Tile Index:**
    For a normalized position $pos.z \in [0, 1]$, calculate the index of the z-tile group:
    $$ thisTileN = \lfloor (N_z - 1.0) \times pos.z \rfloor $$
    $$ thisTileNp1 = \min(thisTileN + 1.0, N_z) $$

3.  **Calculate Spatial Tile Indices:**
    For a given `thisTileN`, calculate the x and y tile indices within the 2D texture grid:
    $$ zTile = \lfloor thisTileN / tilesPerLayer \rfloor $$
    $$ yTile = \lfloor (thisTileN - (zTile \times tilesPerLayer)) / nTiles.x \rfloor $$
    $$ xTile = \text{mod}(thisTileN - (zTile \times tilesPerLayer), nTiles.x) $$

4.  **Calculate Texture Coordinates:**
    Calculate the 2D texture coordinates for the current spatial position within the identified tile:
    $$ thisPoint.x = (xTile + pos.x) \times tileDim.x $$
    $$ thisPoint.y = (yTile + pos.y) \times tileDim.y $$

5.  **Sample Z-Slices:**
    Retrieve the data values from the Red, Green, and Blue channels of the tile at `thisPoint`. Note: The source code only handles $zTile \in \{0, 1, 2\}$ explicitly, implying a limitation or specific implementation for small z-depths or multi-texture handling not fully detailed in the snippet [S1, unofficial].
    $$ datumN = \text{Channel}(texture2D(dataTexture, thisPoint), zTile) $$
    $$ datumNp1 = \text{Channel}(texture2D(dataTexture, thisPoint), zTile + 1) $$
    Where $\text{Channel}(tex, 0) = R$, $\text{Channel}(tex, 1) = G$, $\text{Channel}(tex, 2) = B$.

6.  **Interpolate Z:**
    Calculate the fractional distance within the z-tile:
    $$ zDiff = \text{mod}((N_z - 1.0) \times pos.z, 1.0) $$
    Interpolate the value:
    $$ value = (1.0 - zDiff) \times datumN + zDiff \times datumNp1 $$

7.  **Ray Marching:**
    *   Transform projected coordinates to texture space:
        $$ firstPassTexCoord.x = \frac{(projectedCoords.x / projectedCoords.w) + 1.0}{2.0} $$
        $$ firstPassTexCoord.y = \frac{(projectedCoords.y / projectedCoords.w) + 1.0}{2.0} $$
    *   Get back position:
        $$ backPos = texture2D(firstPassTexture, firstPassTexCoord).xyz $$
    *   Get front position from vertex shader:
        $$ frontPos = worldSpaceCoords $$
    *   Calculate ray direction and length:
        $$ dir = backPos - frontPos $$
        $$ rayLength = \text{length}(dir) $$
    *   Initialize state:
        $$ currentPosition = frontPos $$
        $$ accumulatedColor = (0, 0, 0) $$
        $$ accumulatedAlpha = 0.0 $$
    *   Loop for $i$ from 0 to `steps`:
        *   Sample data:
            $$ sample = getRGBAfromDataTex(dataTexture, currentPosition, dataShape, textureShape) $$
        *   Composite:
            $$ accumulatedColor += (1.0 - accumulatedAlpha) \times sample.rgb \times sample.a $$
            $$ accumulatedAlpha += sample.a $$
        *   Advance:
            $$ delta = 1.0 / steps $$
            $$ deltaDirection = \text{normalize}(dir) \times delta $$
            $$ currentPosition += deltaDirection $$
        *   Break if $accumulatedAlpha \ge 1.0$ or ray length exceeded.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_rgba_from_tiled_2d_texture(data_texture, pos, data_shape, texture_shape):
    """
    Samples a 3D volume stored in a 2D tiled texture with z mapped to RGB channels.
    
    Args:
        data_texture: 2D texture object
        pos: vec3, normalized position in [0,1]
        data_shape: vec3, (Nx, Ny, Nz)
        texture_shape: vec2, (Tx, Ty)
        
    Returns:
        vec4: (r, g, b, a)
    """
    Nx, Ny, Nz = data_shape
    Tx, Ty = texture_shape
    
    # Calculate tile grid dimensions
    n_tiles_x = floor(Tx / Nx)
    n_tiles_y = floor(Ty / Ny)
    tiles_per_layer = n_tiles_x * n_tiles_y
    
    # Calculate tile size in normalized texture coordinates
    tile_dim_x = 1.0 / n_tiles_x
    tile_dim_y = 1.0 / n_tiles_y
    
    # Determine which z-tile group we are in
    # Note: Source uses (Nz-1) for indexing, implying 0-based indexing up to Nz-1
    this_tile_n = floor((Nz - 1.0) * pos.z)
    this_tile_n_p1 = min(this_tile_n + 1.0, float(Nz))
    
    # Calculate x, y tile indices for this z-tile group
    z_tile = floor(this_tile_n / tiles_per_layer)
    y_tile = floor((this_tile_n - (z_tile * tiles_per_layer)) / n_tiles_x)
    x_tile = mod(this_tile_n - (z_tile * tiles_per_layer), n_tiles_x)
    
    # Calculate texture coordinates within the specific tile
    tex_x = (x_tile + pos.x) * tile_dim_x
    tex_y = (y_tile + pos.y) * tile_dim_y
    
    # Sample the tile
    # Note: The source code only handles z_tile 0, 1, 2 explicitly.
    # For z_tile > 2, this implementation would fail or require multiple textures.
    # We assume the source implies a limitation or specific use case.
    # If z_tile > 2, we cannot retrieve data from a single RGB channel mapping 
    # without additional logic not present in the source.
    if z_tile > 2:
        return vec4(0.0, 0.0, 0.0, 0.0) # Placeholder for unsupported depth
        
    tile_sample = sample_2d_texture(data_texture, vec2(tex_x, tex_y))
    
    # Retrieve datumN and datumNp1 based on z_tile
    # Channel 0 = R, 1 = G, 2 = B
    if z_tile == 0:
        datum_n = tile_sample.r
        datum_n_p1 = tile_sample.g
    elif z_tile == 1:
        datum_n = tile_sample.g
        datum_n_p1 = tile_sample.b
    elif z_tile == 2:
        datum_n = tile_sample.b
        datum_n_p1 = 0.0 # Out of bounds for single tile RGB mapping
    else:
        datum_n = 0.0
        datum_n_p1 = 0.0
        
    # Interpolate in Z
    z_diff = mod((Nz - 1.0) * pos.z, 1.0)
    value = (1.0 - z_diff) * datum_n + z_diff * datum_n_p1
    
    # Convert to RGBA
    color = get_datum_color(value) # Implementation specific
    alpha = value * alpha_correction
    
    return vec4(color, alpha)
```

**Edge Cases and Pitfalls:**
*   **Z-Depth Limitation:** The method maps z-slices to RGB channels. A single 2D texture tile can only represent 3 z-slices. If the volume depth $N_z$ is large, this method as described requires either multiple 2D textures or a different indexing scheme not detailed in the source [S1, unofficial]. The source code explicitly checks for `zTile` 0, 1, 2, suggesting it is designed for very shallow volumes or a specific multi-texture setup [S1, unofficial].
*   **Viewing Artifacts:** Viewing rays entering from sides other than the face-on x,y planes cause a "weird haze effect" that blocks the volume [S1, unofficial]. Looking directly along the z-planes (edge-on to tiles) causes rendering to become "really messed up" [S1, unofficial]. These artifacts are likely due to the discrete nature of the tiling and channel mapping causing discontinuities or incorrect sampling when the ray traverses tile boundaries or aligns with channel boundaries.

## Subtopics

### Channel Tiling
A specific variant where the z-dimension is mapped directly to color channels (R, G, B, A) of a 2D texture. It is chosen when hardware lacks 3D texture support and the volume depth is small (<=4 slices per tile). It differs from spatial tiling by packing depth into color rather than spatial coordinates. Link: [Tiled 2D Texture Volume Representation](52-tiled-2d-texture-volume-representation.md).

### Spatial Tiling (Bricking)
A method where the 3D volume is divided into 3D bricks, which are then flattened into 2D textures or stored in a hierarchy. It is chosen for larger volumes where channel mapping is insufficient. It differs from channel tiling by using spatial coordinates for all dimensions. Link: [Window/Level Volume Raycasting with Bricking](2775-window-level-volume-rendering-with-bricking.md).

## Operating characteristics
*   **Time Complexity:** Sampling is $O(1)$ per ray step, but requires multiple texture lookups and arithmetic operations to calculate tile indices and interpolate. Ray marching is $O(N)$ where $N$ is the number of steps.
*   **Memory Complexity:** $O(N_x \times N_y \times N_z)$ for the raw data, but stored in 2D textures. The source uses `vec2(1024, 1024)` tiles, which may pad the data.
*   **Performance:** The source does not provide measured performance metrics. The method is limited by the maximum 2D texture size (e.g., 4096x4096) and the number of available textures if multi-texturing is used for deeper volumes [S1, unofficial].
*   **Strengths:** Compatible with 2D-only texture hardware. Simple indexing logic.
*   **Limitations:** Severe artifacts when viewing from non-face-on angles. Limited depth per tile (3 slices) without complex extensions. Hardcoded dimensions in the source example suggest poor generalizability without modification.
*   **When to use:** Legacy WebGL environments or hardware without 3D texture support, for small-depth volumes.
*   **When not to use:** High-performance rendering, large volumes, or when view-independent quality is required.

## Background
The method originates from a Stack Overflow question discussing WebGL volume rendering artifacts [S1, unofficial]. The author describes tiling 3D data to a 2D texture by tiling laterally and along RGB channels to render a synthetic sphere. The technique is a workaround for the lack of 3D texture support in WebGL 1.0. The source code is provided as a complete shader implementation.

## Sources
[S1] Volume rendering artefacts when looking along the z-plane, https://stackoverflow.com/questions/28604226/volume-rendering-artefacts-when-looking-along-the-z-plane, fetched 2026-10-27 (unofficial)
