# Simple Heightmap Voxel Generation

Also called: Sine Terrain Grid

## Overview
Simple Heightmap Voxel Generation is a procedural method for creating a 2D terrain mesh within a 3D voxel space by calculating vertex heights using sine and cosine functions based on world coordinates [S1, unofficial]. It solves the problem of generating a deterministic, smooth terrain surface for volume rendering or mesh extraction without storing explicit heightmap data [S1, unofficial]. Unlike [Random Heightmap Terrain Generation](5094-random-heightmap-terrain-generation.md), which uses stochastic values, this method produces a continuous, periodic wave pattern [S1, unofficial]. It differs from [Procedural Tree Voxel Generation](4445-procedural-tree-voxel-generation.md) and [Asymmetric Wing Stroke](5094-asymmetric-wing-stroke.md) by focusing on planar, grid-aligned terrain rather than complex 3D structures or organic shapes [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Integer world coordinates $(x, z)$ within a defined grid area, along with fixed frequency and amplitude parameters [S1, unofficial].
- **Output:** A flat float array (vertex buffer) containing triangle vertices for the terrain mesh [S1, unofficial].

**Data Structures:**
- **Vertex Buffer:** A 1D array of floats where every 6 vertices (3 floats each) represent a quad [S1, unofficial]. Each quad corresponds to a unit square in the XZ plane [S1, unofficial]. Each vertex consists of 3 floats representing $(x, y, z)$ coordinates [S1, unofficial]. The buffer is filled sequentially during generation [S1, unofficial].
- **Grid Layout:** The world is divided into square chunks [S1, unofficial]. Each chunk contains $16 \times 16$ vertices [S1, unofficial]. The reference implementation uses a $2 \times 2$ chunk grid, resulting in a $32 \times 32$ world area [S1, unofficial].

**Parameters:**
- **Chunk Size:** 16 vertices per side [S1, unofficial].
- **Frequency Factor:** $0.15$ used in the sine and cosine arguments [S1, unofficial].
- **Amplitude:** $2.0$ multiplier applied to the combined trigonometric result [S1, unofficial].
- **Base Height:** $0.0$ vertical offset [S1, unofficial].

**Algorithm Steps:**
1. Initialize an index counter to 0 for the output buffer [S1, unofficial].
2. Iterate over chunk coordinates `chunkX` from 0 to 1 and `chunkZ` from 0 to 1 [S1, unofficial].
3. For each chunk, iterate over local coordinates `x` from 0 to 15 and `z` from 0 to 15 [S1, unofficial].
4. Calculate world coordinates using the formula:
   $$ \text{worldX} = \text{chunkX} \times 16 + x $$
   $$ \text{worldZ} = \text{chunkZ} \times 16 + z $$
5. Calculate the height $h$ for the current quad using the sine and cosine functions:
   $$ h = 0.0 + (\sin(\text{worldX} \times 0.15) + \cos(\text{worldZ} \times 0.15)) \times 2.0 $$
6. Write the first triangle (indices 0, 1, 2) to the buffer using the calculated height for all corners: Vertex 0 is $(\text{worldX}, h, \text{worldZ})$, Vertex 1 is $(\text{worldX}+1, h, \text{worldZ})$, and Vertex 2 is $(\text{worldX}, h, \text{worldZ}+1)$ [S1, unofficial].
7. Write the second triangle (indices 3, 4, 5) to the buffer using the same height: Vertex 3 is $(\text{worldX}+1, h, \text{worldZ})$, Vertex 4 is $(\text{worldX}+1, h, \text{worldZ}+1)$, and Vertex 5 is $(\text{worldX}, h, \text{worldZ}+1)$ [S1, unofficial].
8. Advance the buffer index by 18 floats (6 vertices) after writing each quad [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def generate_simple_heightmap(chunk_x_max=2, chunk_z_max=2, chunk_size=16, frequency=0.15, amplitude=2.0, base_height=0.0):
    """
    Generates a voxel terrain mesh using sine/cosine height functions.
    
    Args:
        chunk_x_max: Number of chunks along X axis.
        chunk_z_max: Number of chunks along Z axis.
        chunk_size: Vertices per chunk side.
        frequency: Factor for sine/cosine arguments.
        amplitude: Multiplier for height.
        base_height: Base Y offset.
        
    Returns:
        list: Flat list of floats representing vertices (x, y, z) for triangles.
    """
    buffer = []
    
    for chunk_x in range(chunk_x_max):
        for chunk_z in range(chunk_z_max):
            for x in range(chunk_size):
                for z in range(chunk_size):
                    # Calculate world coordinates
                    world_x = chunk_x * chunk_size + x
                    world_z = chunk_z * chunk_size + z
                    
                    # Calculate height based on world coordinates
                    height = base_height + (math.sin(world_x * frequency) + math.cos(world_z * frequency)) * amplitude
                    
                    # Write first triangle (indices 0, 1, 2)
                    buffer.extend([world_x, height, world_z])
                    buffer.extend([world_x + 1, height, world_z])
                    buffer.extend([world_x, height, world_z + 1])
                    
                    # Write second triangle (indices 3, 4, 5)
                    buffer.extend([world_x + 1, height, world_z])
                    buffer.extend([world_x + 1, height, world_z + 1])
                    buffer.extend([world_x, height, world_z + 1])
                    
    return buffer
```

**Edge Cases and Pitfalls:**
- **Flat Quads:** The implementation calculates a single height value for the entire quad based on the bottom-left corner $(\text{worldX}, \text{worldZ})$ [S1, unofficial]. This results in a flat horizontal quad rather than a smoothly interpolated surface, as all four corners share this height [S1, unofficial].
- **Boundary Conditions:** The loop bounds are strict ($0$ to $15$ for local coordinates, $0$ to $1$ for chunks) [S1, unofficial]. No padding or overlap is handled [S1, unofficial].
- **Precision:** The use of integer coordinates for sine/cosine arguments can lead to aliasing if the frequency is high, but the default $0.15$ factor produces a smooth pattern [S1, unofficial].

## Subtopics

### Procedural Voxel Map Generation
Generates a 3D voxel map by applying a procedural color function and randomly placing solid rectangular blocks within defined bounds [S3, unofficial]. Unlike the heightmap method which generates a surface mesh, this method populates a volume with discrete blocks [S3, unofficial]. It uses a deterministic hash for vein generation: $((x \times 17) \oplus (y \times 31) \oplus (z \times 47)) \cdot \text{wrapping\_mul}(0x45d9f3b)$ [S3, unofficial]. It also calculates pulse and swirl components using sine functions: $\text{pulse} = \sin(x^2 + y^2 + z \times 19)$ and $\text{swirl} = \sin((x - y) \times 0.11 + z \times 0.27)$ [S3, unofficial]. RGB channels are computed with a fade factor of $z / 63.0$ [S3, unofficial]. The algorithm initializes a flat map at Z-height 63, loops 2048 times to sample random booleans for solidity, generates random start/end coordinates clamped to `VSID` and `MAXZDIM`, and calls `map.set_rect` with bounds and color set [S3, unofficial]. Parameters include `VSID` (horizontal size), `MAXZDIM` (vertical size), and 2048 random blocks [S3, unofficial].

### Random Heightmap Terrain Generation
Generates a grid mesh with vertices assigned random heights within a specified range [S2, unofficial]. It differs from Simple Heightmap Voxel Generation by using `Math.random()` instead of deterministic sine/cosine functions [S2, unofficial]. The grid size is defined by `xDetail` and `zDetail` segments, with world dimensions `xWidth` and `zWidth` [S2, unofficial]. Vertices are stored in an array of size $(xDetail + 1) \times (zDetail + 1)$ [S2, unofficial]. The world coordinates for a vertex are calculated as:
$$ \text{tempX} = \frac{iX}{xDetail} \times xWidth - \frac{xWidth}{2} $$
$$ \text{tempZ} = \frac{iZ}{zDetail} \times zWidth - \frac{zWidth}{2} $$
$$ \text{tempY} = \text{height} \times (2 \times (\text{Math.random()} - 0.5)) $$
where `height` is the maximum absolute height [S2, unofficial]. Faces are generated using a standard diagonal split, and normals are computed via cross product of edge vectors [S2, unofficial]. The layout stores face objects with vertex indices (A, B, C) and normal components (NX, NY, NZ) [S2, unofficial].

## Operating characteristics
- **Time Complexity:** The algorithm performs constant-time trigonometric and arithmetic operations per quad, resulting in linear time complexity $O(N)$ relative to the number of chunks and local vertices [G].
- **Memory Complexity:** The output requires linear memory $O(N)$ for the vertex buffer [G].
- **Performance:** The method executes extremely fast because it involves no complex noise lookups or interpolation, only basic trigonometry [S1, unofficial]. It is suitable for real-time generation in constrained environments like WebAssembly or mobile devices [S1, unofficial].
- **Strengths:** The method is deterministic and straightforward to implement [S1, unofficial].
- **Limitations:** The resulting terrain is periodic and repetitive, lacking natural variation [S1, unofficial]. The flat-quad implementation results in a non-smooth surface [S1, unofficial]. It is not suitable for generating complex, organic terrain features [S1, unofficial].

## Background
The method originates from a WebAssembly-based Minecraft-like voxel engine implementation [S1, unofficial]. It was designed to provide a simple, deterministic terrain generation technique for a 2D chunk-based system [S1, unofficial]. The specific implementation uses a $2 \times 2$ chunk grid with $16 \times 16$ vertices per chunk, outputting a flat float array for rendering [S1, unofficial]. The approach is a basic example of procedural geometry generation, relying on the periodic nature of sine and cosine functions to create a wave-like terrain [S1, unofficial].

## Sources
[S1] nilabh25bcs10235-beep/minecraft-wasm, https://github.com/nilabh25bcs10235-beep/minecraft-wasm, fetched 2026-10-26 (unofficial)
[S2] mayakeele/bad-minecraft, https://github.com/mayakeele/bad-minecraft, fetched 2026-10-27 (unofficial)
[S3] MuffinTastic/rust-voxlap-vxl, https://github.com/MuffinTastic/rust-voxlap-vxl, fetched 2026-10-28 (unofficial)
