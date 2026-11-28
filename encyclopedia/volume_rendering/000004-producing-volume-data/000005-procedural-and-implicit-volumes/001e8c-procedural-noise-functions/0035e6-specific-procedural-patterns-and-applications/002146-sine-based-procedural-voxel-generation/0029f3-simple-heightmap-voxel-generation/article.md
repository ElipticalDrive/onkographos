# Simple Heightmap Voxel Generation

Also called: Sine Terrain Grid

## Overview
Simple Heightmap Voxel Generation is a procedural method for creating voxel terrain meshes by mapping 2D grid coordinates to heights using deterministic sine and cosine functions [S1, unofficial]. It replaces stochastic noise with deterministic trigonometric waves to ensure spatial continuity and predictable patterns [S1, unofficial]. It differs from its sibling, Random Heightmap Terrain Generation, by using fixed trigonometric evaluation instead of random assignment [S1, unofficial]. It is distinct from its parent, Sine-based procedural voxel generation, by specifically implementing a 2D heightmap approach for terrain meshing rather than general 3D field evaluation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Grid dimensions defined by chunk count (2x2 chunks) and chunk size (16x16 vertices per chunk), resulting in a 32x32 world area [S1, unofficial].
*   **Output:** A flat float array buffer containing vertex data for triangle strips representing the terrain surface [S1, unofficial].

**Data Structures:**
*   **Output Buffer:** A flat float array where every 6 vertices (3 floats each) represent a quad (2 triangles) [S1, unofficial]. The buffer is filled sequentially [S1, unofficial].
*   **Vertex Format:** Each vertex consists of 3 floats: $(x, y, z)$ [S1, unofficial].

**Parameters:**
*   **Chunk Size:** 16 vertices per side [S1, unofficial].
*   **Chunk Grid:** 2x2 chunks [S1, unofficial].
*   **Frequency Factor:** $0.15$ used in trigonometric calculations [S1, unofficial].
*   **Amplitude Multiplier:** $2.0$ applied to the sum of sine and cosine [S1, unofficial].
*   **Base Height Offset:** $0.0$ [S1, unofficial].

**Formulas:**
The height $h$ for a world coordinate $(worldX, worldZ)$ is calculated as [S1, unofficial]:
$$
h = 0.0 + (\sin(worldX \times 0.15) + \cos(worldZ \times 0.15)) \times 2.0
$$
Where $worldX$ and $worldZ$ are integer world coordinates derived from the chunk and local indices [S1, unofficial].

**Steps:**
1.  Initialize an empty output buffer [S1, unofficial].
2.  Iterate over chunk X from 0 to 1 [S1, unofficial].
3.  Iterate over chunk Z from 0 to 1 [S1, unofficial].
4.  Iterate over local X from 0 to 15 [S1, unofficial].
5.  Iterate over local Z from 0 to 15 [S1, unofficial].
6.  Calculate world coordinates: $worldX = chunkX \times 16 + x$ and $worldZ = chunkZ \times 16 + z$ [S1, unofficial].
7.  Calculate height: $height = (\sin(worldX \times 0.15) + \cos(worldZ \times 0.15)) \times 2.0$ [S1, unofficial].
8.  Write the first triangle vertices to the buffer: $(worldX, height, worldZ)$, $(worldX+1, height, worldZ)$, $(worldX, height, worldZ+1)$ [S1, unofficial].
9.  Write the second triangle vertices to the buffer: $(worldX+1, height, worldZ)$, $(worldX+1, height, worldZ+1)$, $(worldX, height, worldZ+1)$ [S1, unofficial].
10. Repeat until all chunks and vertices are processed [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def generate_simple_heightmap(chunk_x_count=2, chunk_z_count=2, chunk_size=16):
    """
    Generates a voxel terrain mesh using sine/cosine heightmap.
    Returns a flat list of floats representing vertices.
    """
    buffer = []
    
    freq = 0.15
    amp = 2.0
    
    for chunk_x in range(chunk_x_count):
        for chunk_z in range(chunk_z_count):
            for x in range(chunk_size):
                for z in range(chunk_size):
                    world_x = chunk_x * chunk_size + x
                    world_z = chunk_z * chunk_size + z
                    
                    height = (math.sin(world_x * freq) + math.cos(world_z * freq)) * amp
                    
                    # Triangle 1: (x, z), (x+1, z), (x, z+1)
                    buffer.append(world_x)
                    buffer.append(height)
                    buffer.append(world_z)
                    
                    buffer.append(world_x + 1)
                    buffer.append(height)
                    buffer.append(world_z)
                    
                    buffer.append(world_x)
                    buffer.append(height)
                    buffer.append(world_z + 1)
                    
                    # Triangle 2: (x+1, z), (x+1, z+1), (x, z+1)
                    buffer.append(world_x + 1)
                    buffer.append(height)
                    buffer.append(world_z)
                    
                    buffer.append(world_x + 1)
                    buffer.append(height)
                    buffer.append(world_z + 1)
                    
                    buffer.append(world_x)
                    buffer.append(height)
                    buffer.append(world_z + 1)
                    
    return buffer
```

**Edge Cases and Pitfalls:**
*   **Flat Quads:** The implementation calculates a single height per quad using the bottom-left corner coordinates and applies it to all four vertices, producing flat-topped quads rather than interpolated slopes [S1, unofficial].
*   **Boundary Conditions:** The grid is strictly bounded by the chunk dimensions with no wrapping or infinite terrain implied [S1, unofficial].

## Subtopics

### [Random Heightmap Terrain Generation](4992-random-heightmap-terrain-generation.md)
Random Heightmap Terrain Generation creates a voxel terrain mesh by assigning independent, uniformly distributed random heights to vertices on a 2D grid. It differs from its parent, Simple Heightmap Voxel Generation, by replacing deterministic sine/cosine functions with stochastic noise, and from its child, Midpoint Displacement Fractal, by lacking spatial correlation between adjacent vertices. This method is chosen when a non-smooth, jagged, or purely random terrain surface is desired without the computational overhead of fractal algorithms.

### Procedural Voxel Map Generation
Procedural Voxel Map Generation creates a 3D voxel map by applying a procedural color function and randomly placing solid rectangular blocks within defined bounds [S3, unofficial]. It defines a color function with deterministic hashing for veins using $((x \times 17) \oplus (y \times 31) \oplus (z \times 47)) \times 0x45d9f3b$, computes pulse $= \sin(x^2 + y^2 + z \times 19)$ and swirl $= \sin((x - y) \times 0.11 + z \times 0.27)$, calculates RGB channels with fade $= z / 63.0$, and computes shade based on vein bits [S3, unofficial]. It creates a flat map at $Z=63$, loops 2048 times sampling random booleans for solidity, generates random start/end coordinates clamped to VSID and MAXZDIM, calls `map.set_rect` with bounds and color set, and writes output via `map.write_columns` [S3, unofficial]. Parameters include VSID (horizontal dimension size), MAXZDIM (vertical dimension size), 63 (initial flat Z height), and 2048 (number of random blocks) [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the total number of quads, with constant-time trigonometric evaluation per quad [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for the output buffer, fixed at $6 \times 1600$ floats for the default 4-chunk configuration (32x32 grid) [S1, unofficial].
*   **Performance:** Very fast due to the simplicity of the height calculation without noise lookup tables or complex interpolation [S1, unofficial].
*   **Strengths:** Deterministic output, easy to implement, low memory footprint, predictable terrain patterns [S1, unofficial].
*   **Limitations:** Limited terrain variety due to periodic patterns, flat-topped quads without interpolation, no 3D internal structure generation [S1, unofficial].

## Background
Simple Heightmap Voxel Generation originates from early web-based voxel game implementations, specifically those targeting WebAssembly for real-time performance [S1, unofficial]. It was developed as a straightforward method to generate terrain meshes in browser environments, leveraging the simplicity of trigonometric functions for height calculation [S1, unofficial]. The method is part of a broader family of procedural generation techniques used in voxel-based games and simulations [S1, unofficial].

## Sources
[S1] nilabh25bcs10235-beep/minecraft-wasm, https://github.com/nilabh25bcs10235-beep/minecraft-wasm, fetched 2026-10-26 (unofficial)
[S2] mayakeele/bad-minecraft, https://github.com/mayakeele/bad-minecraft, fetched 2026-10-27 (unofficial)
[S3] MuffinTastic/rust-voxlap-vxl, https://github.com/MuffinTastic/rust-voxlap-vxl, fetched 2026-10-28 (unofficial)
