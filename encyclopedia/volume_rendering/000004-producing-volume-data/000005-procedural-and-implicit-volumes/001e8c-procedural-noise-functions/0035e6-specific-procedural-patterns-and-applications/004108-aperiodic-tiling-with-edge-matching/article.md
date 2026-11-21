# Aperiodic Tiling with Edge Matching

Also called: Aperiodic mapping, Edge-matching triangular patterns

## Overview
A method to avoid repetitivity when mapping pre-shaded texture cells onto a terrain or volume surface by enforcing aperiodic boundary conditions and edge-matching constraints [S1]. It differs from standard periodic tiling by correlating twin edges to prevent seams, and from noise-based tiling by relying on pre-computed geometric patterns rather than procedural generation at render time [S1]. It builds on [Specific Procedural Patterns and Applications](7615-specific-procedural-patterns-and-applications.md) and contrasts with [Sine-based procedural voxel generation](2477-sine-based-procedural-voxel-generation.md), [Spherical Density Ramp with Noise](2610-spherical-density-ramp-with-noise.md), [Mandelbulb Fractal Generation](3327-mandelbulb-fractal-generation.md), and [Volumetric Density Generation](4894-volumetric-density-generation.md).

## How it works
**Inputs and Outputs:**
*   **Input:** A set of pre-shaded triangular tile patterns (typically 8 triangles: 4 north-pointing, 4 south-pointing) and a spatial coordinate system [S1].
*   **Output:** A seamless, non-repeating texture map for terrain or volume rendering [S1].

**Data Structures:**
*   **Reference Texture:** A 2D image containing the 8 triangular tiles arranged to satisfy edge constraints [S1].
*   **Tile Selection Logic:** A mapping from spatial coordinates to one of the 8 tile types based on aperiodic triplet selection [S1].

**Steps:**
1.  **Define Boundary Conditions:** Assume 2 different boundary conditions in each of the 3 directions of a triangular tiling, yielding 8 possible triplets and 16 possible triangles (pointing north or south) [S1].
2.  **Select Subset:** Choose a subset of 8 triangles (4 north, 4 south) to save memory, ensuring each triplet is used once without breaking aperiodicity [S1].
3.  **Arrange Reference Texture:** Place the 8 tiles in the reference texture such that edge constraints are met, ensuring each edge kind appears twice as "twin edges" [S1].
4.  **Shear and Wrap:** Apply a shear transformation to the pattern to convert the natural slanted wrapping (where bottom and top triangles share an edge) into a square wrapping tractable by hardware textures [S1].
5.  **Build Patterns (Content Placement):** Enforce that objects crossing an edge axis are replicated on the twin edge, making neighborhoods correlated but not identical [S1]. Ensure objects never cross two edges (e.g., corners) to avoid correlating them and replicating on half the cells [S1]. Place trees crossing boundaries first (replicated on twin edges), starting with one tree close to a corner (not crossing 2 edges) to avoid vicinity bias [S1]. Freely place other trees, constrained only by not crossing the triangle tile boundary [S1].

**Formulas:**
*   **Shear Transformation:** $\mathbf{v}' = \mathbf{M}_{shear} \mathbf{v}$, where $\mathbf{v} = [x, y]^T$ is a vertex coordinate, $\mathbf{v}'$ is the transformed coordinate, and $\mathbf{M}_{shear} = \begin{bmatrix} 1 & k \\ 0 & 1 \end{bmatrix}$ with $k$ as the shear factor [G].

**Parameters:**
*   **Number of Tiles:** 8 triangles (4 north, 4 south) are sufficient for aperiodicity. 16 triangles can be used for higher variability [S1].
*   **Texture Resolution:** Not given in the sources, but must support the sheared square wrapping [S1].

**Edge Cases and Pitfalls:**
*   **Hardware Linear Interpolation:** At tile borders, hardware linear interpolation blends outside color with inside color, causing artifacts if the reference texture is not arranged to make twin edges coherent [S1].
*   **Corner Crossing:** Objects crossing two edges (corners) must be avoided, as this correlates edges and causes replication on half the cells [S1].
*   **Pre-shading Limitation:** Patterns are pre-shaded, so they cannot be rotated at render time [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_reference_texture(triangles):
    # 1. Arrange tiles so each edge kind appears twice (twin edges)
    arranged = []
    for tri in triangles:
        twin_idx = find_twin(tri, triangles)
        arranged.append((tri, twin_idx))
    grid = arrange_tiles(arranged)
    # 2. Apply shear transformation to convert slanted wrapping to square
    return apply_shear_and_wrap(grid)

def find_twin(tri, triangles):
    # Returns index of triangle sharing the same boundary condition on the matching edge
    for i, other in enumerate(triangles):
        if i != tri.id and tri.edge_type == other.edge_type:
            return i
    return -1

def arrange_tiles(arranged):
    # Places tiles in a 2D grid so twin edges are adjacent
    grid = [t for t, _ in arranged]
    return grid

def apply_shear_and_wrap(grid):
    # Shear matrix to convert slanted wrapping to square wrapping
    k = 0.5  # Standard shear factor for triangular tiling
    shear = [[1.0, k], [0.0, 1.0]]
    sheared = []
    for tile in grid:
        new_verts = [(shear[0][0]*x + shear[0][1]*y, shear[1][0]*x + shear[1][1]*y) for x, y in tile.verts]
        sheared.append(Tile(verts=new_verts, content=tile.content))
    return pack_to_texture(sheared)

def pack_to_texture(tiles):
    # Combines sheared tiles into a single 2D image buffer
    width = max(t.w for t in tiles)
    height = max(t.h for t in tiles)
    tex = ImageBuffer(width, height)
    for i, t in enumerate(tiles):
        tex.blit(t, offset=(i % 4, i // 4))
    return tex

def sample_aperiodic(u, v, tex, tile_map):
    # Determine which of the 8 triangular tiles covers the current point
    idx = tile_map(u, v)
    tile = tile_map.get_tile(idx)
    # Get local coordinates within the specific triangular tile
    local_u = (u - tile.u_min) / tile.w
    local_v = (v - tile.v_min) / tile.h
    # Sample the texture. Hardware linear interpolation handles coherence.
    return tex.sample(local_u, local_v)
```

## Operating characteristics
*   **Time Complexity:** Constant time per sample, assuming efficient tile lookup. The shearing and wrapping are pre-computed [S1].
*   **Memory Complexity:** Proportional to the number of tiles (8 or 16) and their resolution [S1].
*   **Strengths:** Eliminates visible repetition in large terrains/volumes without requiring complex procedural noise at render time. Compatible with hardware linear filtering if built correctly [S1].
*   **Limitations:** Requires pre-shading, limiting dynamic lighting. Manual placement of objects in tiles is required to avoid artifacts. Limited variability compared to fully procedural methods [S1].
*   **When to Use:** Real-time rendering of large, static or semi-static scenes (e.g., forests) where repetition is noticeable and pre-shading is acceptable [S1].
*   **When Not to Use:** Dynamic lighting scenarios, or when high variability and organic randomness are required without pre-computation [S1].

## Background
Originated in the context of rendering forest scenes in real-time, specifically to avoid repetitivity when mapping texcells onto a terrain [S1]. The method relies on the aperiodic scheme described in [NC99] and is discussed in detail in [S1]. It addresses the limitation of pre-shaded textures, which cannot be rotated, by using separate boundary conditions in each direction [S1]. Scenes are lit by fixed directional lights with precomputed shading stored in the textures [S1].

## Sources
[S1] Rendering Forest Scenes in Real-Time, https://doi.org/10.2312/egwr/egsr04/093-102, fetched 2026-10-26
