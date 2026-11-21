# Simple Heightmap Voxel Generation

Also called: Sine Terrain Grid

## Overview
Simple Heightmap Voxel Generation is a procedural method for creating voxel terrain meshes by mapping 2D grid coordinates to heights using deterministic sine and cosine functions [S1, unofficial]. It solves the problem of generating smooth, periodic terrain surfaces without the computational overhead of fractal noise or the complexity of multi-biome systems. It differs from its sibling, Random Heightmap Terrain Generation, by replacing stochastic noise with deterministic trigonometric waves, ensuring spatial continuity and predictable patterns [S1, unofficial]. It is distinct from its parent, Sine-based procedural voxel generation, by specifically implementing a 2D heightmap approach for terrain meshing rather than general 3D field evaluation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Grid dimensions defined by chunk count (2x2 chunks) and chunk size (16x16 vertices per chunk), resulting in a 32x32 world area [S1, unofficial].
*   **Output:** A flat float array buffer containing vertex data for triangle strips representing the terrain surface [S1, unofficial].

**Data Structures:**
*   **Output Buffer:** A 1D array of floats. Each quad (unit square in the grid) is represented by 6 vertices (3 floats each), forming 2 triangles. The layout is sequential: $[x_0, y_0, z_0, x_1, y_1, z_1, x_2, y_2, z_2, x_3, y_3, z_3, x_4, y_4, z_4, x_5, y_5, z_5]$ [S1, unofficial].
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
Where $worldX$ and $worldZ$ are integer coordinates derived from the chunk and local indices.

**Steps:**
1.  Initialize an index counter `index` to 0.
2.  Iterate over chunk X from 0 to 1.
3.  Iterate over chunk Z from 0 to 1.
4.  Iterate over local X from 0 to 15.
5.  Iterate over local Z from 0 to 15.
6.  Calculate world coordinates:
    $$
    worldX = chunkX \times 16 + x
    $$
    $$
    worldZ = chunkZ \times 16 + z
    $$
7.  Calculate height:
    $$
    height = (\sin(worldX \times 0.15) + \cos(worldZ \times 0.15)) \times 2.0
    $$
8.  Write the first triangle vertices to the buffer at `index`:
    *   Vertex 0: $(worldX, height, worldZ)$
    *   Vertex 1: $(worldX+1, height, worldZ)$
    *   Vertex 2: $(worldX, height, worldZ+1)$
9.  Write the second triangle vertices to the buffer at `index + 9`:
    *   Vertex 3: $(worldX+1, height, worldZ)$
    *   Vertex 4: $(worldX+1, height, worldZ+1)$
    *   Vertex 5: $(worldX, height, worldZ+1)$
10. Increment `index` by 18 (6 vertices $\times$ 3 floats).
11. Repeat until all chunks and vertices are processed.

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
    index = 0
    
    # Pre-calculate frequency and amplitude
    freq = 0.15
    amp = 2.0
    
    for chunk_x in range(chunk_x_count):
        for chunk_z in range(chunk_z_count):
            for x in range(chunk_size):
                for z in range(chunk_size):
                    # Calculate world coordinates
                    world_x = chunk_x * chunk_size + x
                    world_z = chunk_z * chunk_size + z
                    
                    # Calculate height using sine and cosine
                    height = (math.sin(world_x * freq) + math.cos(world_z * freq)) * amp
                    
                    # Define the four corners of the quad
                    # p0: (x, z), p1: (x+1, z), p2: (x, z+1), p3: (x+1, z+1)
                    # Heights are calculated at each corner. 
                    # Note: The source code [S1, unofficial] uses the SAME height for all 4 vertices of the quad.
                    # This creates a flat-topped quad at the calculated height for the bottom-left corner.
                    # This is a specific implementation detail of the "Simple" method.
                    
                    h0 = height
                    h1 = height
                    h2 = height
                    h3 = height
                    
                    # Triangle 1: p0, p1, p2
                    buffer.append(world_x)
                    buffer.append(h0)
                    buffer.append(world_z)
                    
                    buffer.append(world_x + 1)
                    buffer.append(h1)
                    buffer.append(world_z)
                    
                    buffer.append(world_x)
                    buffer.append(h2)
                    buffer.append(world_z + 1)
                    
                    # Triangle 2: p1, p3, p2
                    buffer.append(world_x + 1)
                    buffer.append(h1)
                    buffer.append(world_z)
                    
                    buffer.append(world_x + 1)
                    buffer.append(h3)
                    buffer.append(world_z + 1)
                    
                    buffer.append(world_x)
                    buffer.append(h2)
                    buffer.append(world_z + 1)
                    
                    index += 18
                    
    return buffer
```

**Edge Cases and Pitfalls:**
*   **Flat Quads:** The implementation in [S1, unofficial] calculates the height based on the bottom-left corner $(worldX, worldZ)$ and applies it to all four vertices of the quad. This results in flat-topped quads rather than interpolated slopes, which may appear blocky at higher resolutions or steeper frequencies [S1, unofficial].
*   **Precision:** Using integer coordinates for sine/cosine inputs can lead to aliasing if the frequency factor is not chosen carefully relative to the grid resolution [S1, unofficial].
*   **Boundary Conditions:** The grid is strictly bounded by the chunk dimensions. No wrapping or infinite terrain is implied [S1, unofficial].

## Subtopics

### [Random Heightmap Terrain Generation](4992-random-heightmap-terrain-generation.md)
Random Heightmap Terrain Generation creates a voxel terrain mesh by assigning independent, uniformly distributed random heights to vertices on a 2D grid. It differs from its parent, Simple Heightmap Voxel Generation, by replacing deterministic sine/cosine functions with stochastic noise, and from its child, Midpoint Displacement Fractal, by lacking spatial correlation between adjacent vertices. This method is chosen when a non-smooth, jagged, or purely random terrain surface is desired without the computational overhead of fractal algorithms.

### Procedural Voxel Map Generation
Procedural Voxel Map Generation creates a 3D voxel map by applying a procedural color function and randomly placing solid rectangular blocks within defined bounds [S3, unofficial]. It differs from Simple Heightmap Voxel Generation by operating in 3D space with discrete block placement rather than a continuous 2D heightmap mesh, and by using deterministic hashing for color/vein generation instead of trigonometric height mapping [S3, unofficial]. It is chosen when generating complex internal structures like veins or ores within a solid volume rather than a surface terrain.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the total number of vertices in the grid. The calculation per vertex is constant time, involving two trigonometric functions and basic arithmetic [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for the output buffer. The buffer size is fixed at $6 \times 1600$ floats for the default 4-chunk configuration (32x32 grid), as each quad generates 6 vertices [S1, unofficial].
*   **Performance:** Very fast due to the simplicity of the height calculation. No noise lookup tables or complex interpolation are required [S1, unofficial].
*   **Strengths:** Deterministic output, easy to implement, low memory footprint, predictable terrain patterns [S1, unofficial].
*   **Limitations:** Limited terrain variety (periodic patterns), flat-topped quads if not interpolated, no 3D internal structure [S1, unofficial].
*   **When to use:** Ideal for simple, repetitive terrain backgrounds or when deterministic, reproducible terrain is required [S1, unofficial].
*   **When not to use:** When natural-looking, non-periodic terrain is needed, or when 3D internal voxel structures are required [S1, unofficial][S3, unofficial].

## Background
Simple Heightmap Voxel Generation originates from early web-based voxel game implementations, specifically those targeting WebAssembly for performance [S1, unofficial]. It was developed as a straightforward method to generate terrain meshes in real-time within browser environments, leveraging the simplicity of trigonometric functions for height calculation [S1, unofficial]. The method is part of a broader family of procedural generation techniques used in voxel-based games and simulations [S1, unofficial].

## Sources
[S1] nilabh25bcs10235-beep/minecraft-wasm, https://github.com/nilabh25bcs10235-beep/minecraft-wasm, fetched 2026-10-26 (unofficial)
[S2] mayakeele/bad-minecraft, https://github.com/mayakeele/bad-minecraft, fetched 2026-10-27 (unofficial)
[S3] MuffinTastic/rust-voxlap-vxl, https://github.com/MuffinTastic/rust-voxlap-vxl, fetched 2026-10-28 (unofficial)
