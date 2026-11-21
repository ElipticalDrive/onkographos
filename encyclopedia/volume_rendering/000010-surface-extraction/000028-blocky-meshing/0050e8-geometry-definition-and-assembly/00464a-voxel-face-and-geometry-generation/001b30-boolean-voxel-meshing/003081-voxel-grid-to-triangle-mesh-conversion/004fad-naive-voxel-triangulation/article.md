# Naive Voxel Triangulation
Also called: Full Voxel Cube Rendering
## Overview
Naive Voxel Triangulation is a surface extraction method that renders every voxel in a grid as a complete, closed cube by generating triangles for all six of its faces, regardless of whether the faces are occluded by neighboring voxels [S1, unofficial]. It solves the problem of converting discrete voxel data into a renderable triangle mesh with maximum simplicity, serving as the baseline for more optimized approaches like [Range Grid Triangulation](7146-range-grid-triangulation.md) which selectively generate only exposed faces.
## How it works
**Inputs and outputs:**
- Input: A set of $N$ voxels, where each voxel $v_i$ is defined by its integer grid coordinates $(x_i, y_i, z_i)$ and associated attributes (e.g., color).
- Output: A triangle mesh consisting of $12N$ triangles.

**Data structures:**
The method does not require complex adjacency structures. It iterates over the list of occupied voxels. For each voxel, the geometry is defined by its 8 corners and 6 faces.

**Steps:**
1. For each voxel $v$ at position $(x, y, z)$:
2. Define the 8 vertices of the cube. Assuming unit-sized voxels aligned to the integer grid, the vertices are at $(x, y, z)$ to $(x+1, y+1, z+1)$.
3. Define the 6 faces of the cube. Each face is a square in 3D space.
4. For each of the 6 faces, generate 2 triangles.
   - A square face with corners $A, B, C, D$ (ordered counter-clockwise when viewed from outside) is split into two triangles, e.g., $(A, B, C)$ and $(A, C, D)$.
5. Append the generated triangles to the output mesh.

**Formulas:**
- Triangles per voxel: $T_v = 12$ [S1, unofficial].
- Total triangles: $T_{total} = 12 \times N$ [S1, unofficial].

**Parameters:**
- Voxel size: Typically 1 unit in grid coordinates.
- Face winding: Consistent outward-facing normal convention is required for correct lighting, though not strictly required for visibility in opaque rendering.

**Edge cases and pitfalls:**
- **Occlusion:** The primary pitfall is rendering a massive number of hidden triangles. For a model with 226,036 voxels, this results in 2,712,432 triangles, most of which are internal and never visible [S1, unofficial].
- **Precision:** Integer coordinates are sufficient for vertex positions, avoiding floating-point errors in grid alignment.
- **Degenerate input:** Empty voxels (if present in the input list) should be skipped or handled as empty cubes; the method assumes the input list contains only occupied voxels.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def naive_voxel_triangulation(voxels):
    """
    Generates a triangle mesh by rendering every voxel as a full cube.
    
    Args:
        voxels: List of tuples (x, y, z, color), where x, y, z are integers.
        
    Returns:
        triangles: List of tuples (v1, v2, v3), where each v is a 3D coordinate tuple.
    """
    triangles = []
    
    for x, y, z, color in voxels:
        # Define the 8 corners of the voxel cube
        # Bottom face (z)
        v0 = (x, y, z)
        v1 = (x + 1, y, z)
        v2 = (x + 1, y + 1, z)
        v3 = (x, y + 1, z)
        # Top face (z + 1)
        v4 = (x, y, z + 1)
        v5 = (x + 1, y, z + 1)
        v6 = (x + 1, y + 1, z + 1)
        v7 = (x, y + 1, z + 1)
        
        # Generate 2 triangles per face (6 faces total)
        # Front face (y=0 relative to voxel, normal -y)
        triangles.append((v0, v1, v2))
        triangles.append((v0, v2, v3))
        
        # Back face (y=1 relative to voxel, normal +y)
        triangles.append((v4, v6, v5))
        triangles.append((v4, v7, v6))
        
        # Left face (x=0 relative to voxel, normal -x)
        triangles.append((v0, v3, v7))
        triangles.append((v0, v7, v4))
        
        # Right face (x=1 relative to voxel, normal +x)
        triangles.append((v1, v5, v6))
        triangles.append((v1, v6, v2))
        
        # Bottom face (z=0 relative to voxel, normal -z)
        triangles.append((v0, v2, v1))
        triangles.append((v0, v3, v2))
        
        # Top face (z=1 relative to voxel, normal +z)
        triangles.append((v4, v5, v6))
        triangles.append((v4, v6, v7))
        
    return triangles
```
## Subtopics
## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of voxels. Each voxel generates a constant number of triangles (12).
- **Memory Complexity:** $O(N)$ for the output mesh. The input can be processed in a stream if memory is constrained, but the output size is large.
- **Measured Performance:** For a model with 226,036 voxels, this method produces 2,712,432 triangles [S1, unofficial]. This is significantly higher than necessary; optimized methods that only render exposed faces can reduce this to ~138,260 triangles (5% of the original) for the same model [S1, unofficial].
- **Strengths:** Extremely simple to implement; no neighbor lookups or adjacency checks required; robust to any voxel configuration.
- **Limitations:** Massive over-rendering of occluded geometry; poor performance for dense voxel grids; high memory bandwidth usage due to large triangle counts.
- **When to use:** Prototyping; very sparse voxel grids where most faces are exposed; educational purposes to demonstrate the baseline cost of voxel rendering.
- **When not to use:** Production rendering of dense models; any scenario where performance or memory is a constraint.
## Background
The method is described in "Voxel Rendering Techniques" by Fogleman, which highlights the naive approach of rendering all six faces of every voxel as a starting point for optimization [S1, unofficial]. The example uses a voxel model by @ephtracy (of MagicaVoxel fame) consisting of 226,036 voxels to illustrate the inefficiency of this approach.
## Sources
[S1] Voxel Rendering Techniques, https://medium.com/@fogleman/voxel-rendering-techniques-fa8d869457ca, fetched 2026-10-27 (unofficial)
