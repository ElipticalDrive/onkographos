# Terrain Chunk Voxel Generation

Also called: Perlin Noise Heightmap Generation, Voxel Chunk Generation

## Overview
Terrain Chunk Voxel Generation populates a 3D voxel grid with solid or air blocks based on a 2D Perlin noise heightmap [S1, S2, unofficial]. It distinguishes itself from sibling methods by explicitly handling both surface terrain (for $y \ge 0$) and subsurface layers (for $y < 0$) within the same chunk generation pass, whereas alternatives like [Procedural Cave Height and Layer Generation](3202-procedural-cave-height-and-layer-generation.md) focus on internal void structures rather than surface elevation [S2, unofficial].

## How it works
**Inputs and Outputs**
- **Input**: Chunk position $(C_x, C_y, C_z)$ in chunk coordinates; chunk dimensions $WIDTH$ (x/z) and $HEIGHT$ (y); noise parameters (frequency, octaves, dilation, big number); colorist/texture map for subsurface layers.
- **Output**: A 3D array `voxels[WIDTH][HEIGHT][WIDTH]` where each cell contains a `Voxel` type (e.g., `Solid` or `Air`) and associated data (color, density).

**Data Structures**
- `voxels`: 3D array of size $WIDTH \times HEIGHT \times WIDTH$.
- `Voxel`: Struct containing type enum (`Solid`, `Air`), position, and metadata (e.g., color).
- `dilation`: Vector3 $(d_x, d_y, d_z)$ scaling noise coordinates and height.
- `BIG_NUMBER`: Constant offset for noise domain shifting.

**Parameters**
- $WIDTH$: Chunk width in blocks (e.g., 16, 32).
- $HEIGHT$: Chunk height in blocks (e.g., 64, 128).
- $m\_frequency$: Noise frequency constant, typically $0.01$ [S2, unofficial].
- $nbOctaves$: Number of noise octaves, typically $3$ [S2, unofficial].
- $dilation$: Scaling factors for noise $(x, z)$ and height $(y)$ [S1, unofficial].
- $BIG\_NUMBER$: Constant added to noise coordinates to shift sampling domain [S1, unofficial].

**Steps**
1. **Iterate Chunk Coordinates**: Loop over local chunk coordinates $i$ (x-axis) and $k$ (z-axis) from $0$ to $WIDTH-1$.
2. **Calculate World Coordinates**: Compute world coordinates for the current column:
   $$ x = C_x \cdot WIDTH + i $$
   $$ z = C_z \cdot WIDTH + k $$
3. **Determine Surface Height**: Calculate the surface height $y_{surf}$ using the Perlin noise function:
   $$ y_{surf} = \lfloor d_y \cdot \frac{Perlin2D(x \cdot d_x + BIG\_NUMBER, z \cdot d_z + BIG\_NUMBER) + 1}{2} \rfloor + 1 $$
   Where $Perlin2D$ returns a value in $[-1, 1]$ [S1, unofficial].
4. **Fill Surface Terrain ($y \ge 0$)**:
   - If the chunk's vertical position $C_y \ge 0$ and is within the surface range:
     - For each vertical level $j$ from $0$ to $HEIGHT-1$:
       - If $j \le y_{surf}$, set `voxels[i][j][k]` to `Solid` with color from `colorist` [S1, unofficial].
       - Else, set `voxels[i][j][k]` to `Air` [S1, unofficial].
5. **Fill Subsurface Terrain ($y < 0$)**:
   - If the chunk's vertical position $C_y < 0$:
     - Iterate over all $i$ (x), $j$ (y), and $k$ (z) within the chunk.
     - Compute world coordinates: $x_w = C_x \cdot WIDTH + i$, $y_w = C_y \cdot HEIGHT + j$, $z_w = C_z \cdot WIDTH + k$.
     - Set `voxels[i][j][k]` to the result of `getGroundAt(voxels.cubic_size(), x_w, y_w, z_w)` [S2, unofficial]. This function likely uses noise to determine subsurface material type.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_terrain_chunk(chunk_pos, width, height, dilation, big_number, frequency, octaves, colorist):
    """
    Generates a voxel chunk based on Perlin noise heightmap.
    
    Args:
        chunk_pos: Tuple (cx, cy, cz) - chunk coordinates
        width: int - chunk width/depth
        height: int - chunk height
        dilation: Tuple (dx, dy, dz) - noise and height scaling
        big_number: float - noise domain shift
        frequency: float - noise frequency
        octaves: int - noise octaves
        colorist: Function/Map - returns color for solid voxels
        
    Returns:
        list: 3D list of Voxel objects [width][height][width]
    """
    voxels = [[None] * width for _ in range(height)]
    for j in range(height):
        for k in range(width):
            voxels[j][k] = [None] * width

    cx, cy, cz = chunk_pos

    # Handle surface terrain (y >= 0)
    if cy >= 0:
        for i in range(width):
            for k in range(width):
                # Calculate world coordinates for this column
                x = cx * width + i
                z = cz * width + k
                
                # Calculate height using Perlin noise
                # Perlin2D returns value in [-1, 1]
                noise_val = perlin2d(x * dilation[0] + big_number, 
                                     z * dilation[2] + big_number, 
                                     frequency, octaves)
                
                # Normalize to [0, 1] and scale
                normalized_noise = (noise_val + 1.0) / 2.0
                y_surf = int(dilation[1] * normalized_noise) + 1
                
                for j in range(height):
                    if j <= y_surf:
                        voxels[j][i][k] = Voxel(type="Solid", color=colorist(x, j, z))
                    else:
                        voxels[j][i][k] = Voxel(type="Air")
    
    # Handle subsurface terrain (y < 0)
    else:
        for i in range(width):
            for j in range(height):
                for k in range(width):
                    x = cx * width + i
                    y = cy * height + j
                    z = cz * width + k
                    
                    # Use getGroundAt to determine subsurface material
                    # This function likely uses noise to select material type
                    voxel_type = get_ground_at(width, x, y, z)
                    voxels[j][i][k] = Voxel(type=voxel_type)

    return voxels

def perlin2d(x, z, frequency, octaves):
    """
    Simplified Perlin noise generator.
    Returns value in [-1, 1].
    """
    # Implementation details of Perlin noise are assumed to be available
    # from the parent article [Terrain Generation via Noise](2412-terrain-generation-via-noise.md)
    return noise(x * frequency, z * frequency, octaves)

def get_ground_at(chunk_size, x, y, z):
    """
    Determines the ground type at world coordinates (x, y, z).
    Returns a string or enum representing the voxel type.
    """
    # Implementation depends on specific subsurface layering logic
    # Typically uses additional noise functions for depth-based material selection
    return "Stone" # Placeholder for actual logic
```

**Edge Cases and Pitfalls**
- **Precision**: Floating-point noise values must be correctly normalized to $[0, 1]$ before scaling. Incorrect normalization leads to incorrect height calculations [S1, unofficial].
- **Boundaries**: Ensure chunk coordinates are correctly mapped to world coordinates to avoid gaps or overlaps between adjacent chunks [S1, S2, unofficial].
- **Subsurface Logic**: The `getGroundAt` function is not fully detailed in the sources. Implementers must define the logic for subsurface material selection, which may involve additional noise functions or depth thresholds [S2, unofficial].

## Operating characteristics
- **Time Complexity**: $O(WIDTH^2 \cdot HEIGHT)$ per chunk, as every voxel is visited and potentially computed [S1, S2, unofficial].
- **Memory Complexity**: $O(WIDTH^2 \cdot HEIGHT)$ to store the voxel grid [S1, S2, unofficial].
- **Performance**: Performance is dominated by noise generation and voxel assignment. Efficient noise implementations (e.g., using precomputed gradients) are critical [S1, unofficial].
- **Strengths**: Simple to implement; generates coherent terrain surfaces; handles both surface and subsurface in one pass [S1, S2, unofficial].
- **Limitations**: Limited to heightmap-based terrain; cannot generate overhangs or caves without additional logic (unlike [Procedural Cave Height and Layer Generation](3202-procedural-cave-height-and-layer-generation.md)) [S2, unofficial].
- **When to Use**: Ideal for flat or rolling terrain games where performance is critical and complex cave systems are not required [S1, S2, unofficial].

## Background
Terrain Chunk Voxel Generation is a common technique in voxel-based games and simulations. It relies on 2D Perlin noise to generate heightmaps, which are then extruded into 3D voxel grids [S1, S2, unofficial]. The method was popularized by early voxel engines and remains a standard approach for procedural terrain generation due to its simplicity and efficiency [S1, S2, unofficial].

## Sources
[S1] aaasen/voxel-party, https://github.com/aaasen/voxel-party, fetched 2026-10-27 (unofficial)
[S2] Lensimax/VoxelEngine: Voxel Engine is game ..., https://github.com/Lensimax/VoxelEngine, fetched 2026-10-27 (unofficial)
