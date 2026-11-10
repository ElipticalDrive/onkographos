# Aperiodic Tiling with Edge Matching

Also called: Aperiodic mapping, Edge-matching triangular patterns

## Overview
A method to map pre-shaded textures onto terrain or volume surfaces without visible repetition artifacts by using aperiodic tiling schemes with specific boundary conditions [S1]. Unlike standard periodic wrapping which creates obvious seams, this method ensures that objects crossing tile boundaries are correctly replicated on "twin" edges while preventing correlation artifacts at corners [S1]. It is distinct from sibling methods like [Sine-based procedural voxel generation](2477-sine-based-procedural-voxel-generation.md) because it relies on pre-computed, static texture patterns rather than on-the-fly mathematical noise functions [S1].

## How it works
**Inputs:**
- Spatial coordinates $(u, v, w)$ for texture lookup.
- Reference texture containing the arranged aperiodic tiles.
- Hardware texture sampler supporting linear interpolation.

**Outputs:**
- Color/density value at the sampled coordinate.

**Data Structures:**
- **Reference Texture:** A 2D image containing 8 triangular tiles arranged to satisfy edge constraints.
- **Tile Geometry:** Triangular regions within the texture. Each tile has edges with specific boundary conditions (colors/types).
- **Twin Edges:** Pairs of edges sharing the same boundary condition. Objects crossing one must be replicated on the other.

**Steps:**

1.  **Define Boundary Conditions:**
    Assume 2 different boundary conditions in each of the 3 directions (triangular tiling). This yields 8 possible triplets and 16 possible triangles (pointing north or south) [S1].

2.  **Select Tile Subset:**
    To save memory, select a subset of 8 triangles (4 pointing north, 4 pointing south) to ensure each triplet is used once without breaking aperiodicity [S1]. Higher variability can be achieved by keeping all 16 tiles [S1].

3.  **Arrange Tiles in Reference Texture:**
    Arrange the 8 tiles in the reference texture such that edge constraints are met. Each edge kind appears twice (twin edges) [S1].

4.  **Shear and Wrap:**
    The natural wrapping of triangular tiles is slanted (e.g., bottom and top triangles share an edge). To make it tractable by hardware textures, shear and wrap the pattern to obtain square wrapping [S1].

5.  **Build Patterns (Pre-computation):**
    - Place objects crossing boundary edges first. These objects must be replicated on the twin edge [S1].
    - Ensure neighborhoods of twin edges are correlated but not identical (only content exactly on the axis is identical) [S1].
    - **Constraint:** Objects must never cross two edges (e.g., corners) to avoid correlating them and replicating on half the cells [S1].
    - Start by placing one tree close to a corner (not crossing 2 edges) to avoid vicinity bias [S1].
    - Freely place other trees, constrained only by not crossing the triangle tile boundary [S1].

6.  **Sample Texture:**
    Use standard hardware texture sampling with linear interpolation on the sheared, square-wrapped reference texture [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_aperiodic_texture(u, v, reference_texture):
    """
    Samples the aperiodic texture at coordinates (u, v).
    Assumes the reference texture has been pre-sheared and arranged
    according to the edge-matching constraints.
    """
    # 1. Normalize coordinates to [0, 1]
    u = u % 1.0
    v = v % 1.0
    
    # 2. Sample the texture using hardware linear interpolation
    # The texture is arranged such that edge artifacts are minimized
    # by coherent blending of twin edges.
    color = reference_texture.sample_linear(u, v)
    
    return color

def build_aperiodic_reference_texture():
    """
    Pre-computation step to create the reference texture.
    This is done offline, not per-frame.
    """
    # 1. Define 8 triangular tiles with specific boundary conditions
    tiles = create_triangular_tiles(num_tiles=8)
    
    # 2. Arrange tiles in a 2D grid, ensuring twin edges match
    # Twin edges share the same boundary condition and are placed
    # such that linear interpolation blends them coherently.
    arranged_texture = arrange_tiles_in_grid(tiles)
    
    # 3. Shear the texture to convert slanted wrapping to square wrapping
    sheared_texture = shear_and_wrap(arranged_texture)
    
    # 4. Populate tiles with objects (e.g., trees)
    # - Place objects crossing boundaries first, replicating on twin edges
    # - Ensure no object crosses two edges (corners)
    # - Place one object near a corner (not crossing 2 edges) to avoid bias
    populate_tiles_with_objects(sheared_texture)
    
    return sheared_texture
```

**Parameters:**
- Number of tiles: 8 triangles used in implementation. 16 possible if higher variability required [S1].

**Edge Cases and Pitfalls:**
- **Hardware Linear Interpolation:** At tile borders, hardware linear interpolation blends outside color with inside color. If not coherent, this causes artifacts [S1].
- **Corner Crossing:** Objects crossing two edges (corners) correlate them and replicate on half the cells, breaking aperiodicity [S1].
- **Pre-shaded Limitation:** Patterns cannot be rotated if textures are pre-shaded, limiting dynamic lighting variations [S1].

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample, as it relies on standard texture lookup [S1].
- **Memory Complexity:** Depends on the number of tiles and texture resolution. Using 8 tiles saves memory compared to 16 [S1].
- **Strengths:** Avoids visible repetition in large terrains; compatible with pre-shaded textures [S1].
- **Limitations:** Limited to pre-shaded patterns (no dynamic lighting); requires careful pre-computation of tile arrangements [S1].
- **When to Use:** When rendering large terrains or volumes with static, pre-shaded textures where repetition is undesirable [S1].
- **When Not to Use:** When dynamic lighting or rotation of patterns is required [S1].

## Background
Originated in the context of rendering forest scenes in real-time, specifically to avoid repetitivity when mapping texcells onto terrain [S1]. The method builds on the aperiodic scheme described in [NC99] and addresses issues related to pre-shaded textures as in [CSHD03] [S1]. It is used in real-time rendering of natural scenes, particularly forests [S1].

## Sources
[S1] Rendering Forest Scenes in Real-Time, https://doi.org/10.2312/egwr/egsr04/093-102, fetched 2026-10-26
