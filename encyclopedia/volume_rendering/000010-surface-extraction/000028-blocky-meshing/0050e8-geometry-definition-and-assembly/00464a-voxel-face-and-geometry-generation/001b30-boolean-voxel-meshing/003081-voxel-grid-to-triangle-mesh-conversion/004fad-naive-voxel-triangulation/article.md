# Naive Voxel Triangulation
Also called: Full Voxel Cube Rendering
## Overview
Naive Voxel Triangulation renders every voxel in a grid as a complete, closed cube by generating triangles for all six of its faces, regardless of visibility [S1, unofficial]. It provides a straightforward baseline for converting discrete voxel grids into triangle meshes before applying occlusion culling optimizations [G].
## How it works
**Inputs and outputs:**
- Input: A list of occupied voxels, each defined by integer grid coordinates $(x, y, z)$ and attributes such as color [G].
- Output: A triangle mesh containing $12N$ triangles for $N$ voxels [G].

**Data structures:**
The method requires no adjacency structures or spatial acceleration data structures [G]. It processes voxels sequentially [G].

**Steps:**
1. Iterate over each voxel $v$ at grid coordinates $(x, y, z)$.
2. Compute the 8 corner vertices of the unit cube spanning from $(x, y, z)$ to $(x+1, y+1, z+1)$ [G].
3. Define the 6 square faces of the cube [G].
4. For each face, generate 2 triangles using a consistent winding order for outward-facing normals [G].
5. Append the generated triangles to the output mesh.

**Formulas:**
- Triangles per voxel: $T_v = 12$ [S1, unofficial].
- Total triangles: $T_{\text{total}} = 12 \times N$ [S1, unofficial].

**Parameters:**
- Voxel size: 1 unit in grid coordinates [G].
- Face winding: Consistent outward-facing normal convention is required for correct lighting [G].

**Edge cases and pitfalls:**
- **Occlusion:** The primary pitfall is rendering a massive number of hidden triangles. For a model with 226,036 voxels, this results in 2,712,432 triangles, many of which are not visible [S1, unofficial].
- **Precision:** Integer coordinates are sufficient for vertex positions, avoiding floating-point alignment errors [G].
- **Degenerate input:** The method assumes the input list contains only occupied voxels; empty voxels should be filtered beforehand [G].

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
    for x, y, z, _ in voxels:
        # Define the 8 corners of the voxel cube
        v0 = (x, y, z)
        v1 = (x + 1, y, z)
        v2 = (x + 1, y + 1, z)
        v3 = (x, y + 1, z)
        v4 = (x, y, z + 1)
        v5 = (x + 1, y, z + 1)
        v6 = (x + 1, y + 1, z + 1)
        v7 = (x, y + 1, z + 1)

        # Generate 2 triangles per face (6 faces total) with outward normals
        # -X face
        triangles.append((v0, v4, v3))
        triangles.append((v0, v7, v4))
        # +X face
        triangles.append((v1, v2, v5))
        triangles.append((v1, v6, v2))
        # -Y face
        triangles.append((v0, v5, v1))
        triangles.append((v0, v4, v5))
        # +Y face
        triangles.append((v3, v2, v6))
        triangles.append((v3, v6, v7))
        # -Z face
        triangles.append((v0, v2, v1))
        triangles.append((v0, v3, v2))
        # +Z face
        triangles.append((v4, v6, v5))
        triangles.append((v4, v7, v6))

    return triangles
```
## Subtopics
## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of voxels, as each voxel generates a constant 12 triangles [G].
- **Memory Complexity:** $O(N)$ for the output mesh [G].
- **Measured Performance:** For a model with 226,036 voxels, this method produces 2,712,432 triangles [S1, unofficial]. Optimized methods that only render exposed faces reduce this to 138,260 triangles (5% of the original) for the same model [S1, unofficial].
- **Strengths:** Extremely simple to implement with no neighbor lookups or adjacency checks required [G].
- **Limitations:** Massive over-rendering of occluded geometry and high memory bandwidth usage due to large triangle counts [G].
- **When to use:** Prototyping, educational demonstrations of baseline voxel rendering costs, or very sparse grids where most faces are exposed [G].
- **When not to use:** Production rendering of dense models or performance-constrained scenarios [G].
## Background
The method is documented in a blog post by Fogleman, which uses a 226,036-voxel model by @ephtracy (of MagicaVoxel fame) to demonstrate the baseline triangle count before optimization [S1, unofficial]. It serves as the foundational reference for subsequent voxel meshing optimizations [G].
## Sources
[S1] Voxel Rendering Techniques, https://medium.com/@fogleman/voxel-rendering-techniques-fa8d869457ca, fetched 2026-10-27 (unofficial)
