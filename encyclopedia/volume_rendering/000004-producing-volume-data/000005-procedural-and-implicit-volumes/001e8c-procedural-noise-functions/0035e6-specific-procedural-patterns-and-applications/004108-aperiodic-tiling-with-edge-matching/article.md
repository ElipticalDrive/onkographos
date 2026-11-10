# Aperiodic Tiling with Edge Matching

Also called: Aperiodic mapping, Edge-matching triangular patterns

## Overview
A method to map pre-shaded texture cells onto a terrain or volume surface without visible repetition, using aperiodic boundary conditions and edge-matching constraints. It differs from standard periodic tiling by enforcing specific edge correlations to prevent seams, and from simple noise-based tiling by relying on pre-computed geometric patterns rather than procedural generation at render time.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of pre-shaded triangular tile patterns (typically 8 triangles: 4 north-pointing, 4 south-pointing) arranged in a reference texture.
*   **Output:** A seamless, non-repeating texture map for terrain or volume rendering.

**Data Structures:**
*   **Reference Texture:** A 2D image containing the 8 triangular tiles. To support hardware linear interpolation at boundaries, the texture must be wrapped such that "twin edges" (edges sharing the same boundary condition) are coherent.
*   **Tile Selection Logic:** A mapping from spatial coordinates to one of the 8 tile types based on aperiodic triplet selection.

**Steps:**
1.  **Define Boundary Conditions:** Assume 2 different boundary conditions in each of the 3 directions of a triangular tiling. This yields 8 possible triplets and 16 possible triangles (pointing north or south).
2.  **Select Subset:** Choose a subset of 8 triangles (4 north, 4 south) to save memory. Ensure each triplet is used once to maintain aperiodicity [S1].
3.  **Arrange Reference Texture:**
    *   Place the 8 tiles in the reference texture such that edge constraints are met.
    *   Each edge kind appears twice in the texture; these are "twin edges."
    *   The natural wrapping is slanted (e.g., bottom and top triangles share an edge).
    *   Apply a shear transformation to the pattern to convert the slanted wrapping into a square wrapping tractable by hardware textures [S1].
4.  **Build Patterns (Content Placement):**
    *   **Constraint 1:** Objects crossing an edge axis must be replicated on the twin edge. Neighborhoods are correlated but not identical [S1].
    *   **Constraint 2:** Objects must never cross two edges (e.g., corners) to avoid correlating them and replicating on half the cells [S1].
    *   **Placement Strategy:**
        1.  Place trees crossing boundaries first (replicated on twin edges).
        2.  Start with one tree close to a corner (not crossing 2 edges) to avoid vicinity bias [S1].
        3.  Freely place other trees, constrained only by not crossing the triangle tile boundary [S1].
5.  **Render:** Sample the reference texture using the aperiodic mapping. Use hardware linear interpolation for mag/min filters.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_aperiodic_texture(u, v, reference_texture, tile_map):
    """
    Samples the aperiodic texture at normalized coordinates (u, v).
    reference_texture: The sheared, square-wrapped texture containing 8 tiles.
    tile_map: A function or lookup that determines which of the 8 tiles 
              corresponds to the current spatial location based on aperiodic rules.
    """
    # Determine which of the 8 triangular tiles covers the current point
    tile_index = tile_map(u, v)
    
    # Get the local coordinates within the specific triangular tile
    # Note: The reference texture is sheared, so standard UV mapping applies 
    # to the square-wrapped result.
    local_u, local_v = get_local_coords_in_tile(tile_index, u, v)
    
    # Sample the texture. Hardware linear interpolation handles the 
    # coherence between twin edges if the reference texture was built correctly.
    color = reference_texture.sample(local_u, local_v)
    
    return color

def build_reference_texture(tiles):
    """
    Constructs the reference texture from 8 pre-shaded triangular tiles.
    Ensures twin edges are coherent and applies shear for square wrapping.
    """
    # 1. Arrange tiles so each edge kind appears twice (twin edges)
    arranged_tiles = arrange_tiles_for_twin_edges(tiles)
    
    # 2. Apply shear transformation to convert slanted wrapping to square
    sheared_texture = apply_shear_and_wrap(arranged_tiles)
    
    return sheared_texture

def place_objects_in_tile(tile, objects):
    """
    Places objects (e.g., trees) into a tile respecting edge constraints.
    """
    # Place objects crossing boundaries first
    boundary_objects = [obj for obj in objects if obj.crosses_boundary()]
    for obj in boundary_objects:
        # Replicate on twin edge
        replicate_on_twin_edge(obj)
        
    # Place one tree near a corner (not crossing 2 edges) to avoid bias
    corner_tree = find_near_corner_object(objects)
    if corner_tree:
        place(corner_tree, near_corner=True)
        
    # Place remaining objects freely (not crossing tile boundary)
    free_objects = [obj for obj in objects if not obj.crosses_boundary() and obj != corner_tree]
    for obj in free_objects:
        place(obj, constraint="not_crossing_tile_boundary")
```

**Parameters:**
*   **Number of Tiles:** 8 triangles (4 north, 4 south) are sufficient for aperiodicity. 16 triangles can be used for higher variability [S1].
*   **Texture Resolution:** Depends on implementation, but must support the sheared square wrapping [S1].

**Edge Cases and Pitfalls:**
*   **Hardware Linear Interpolation:** At tile borders, hardware linear interpolation blends outside color with inside color. If the reference texture is not arranged to make twin edges coherent, artifacts will appear [S1].
*   **Corner Crossing:** Objects crossing two edges (corners) must be avoided, as this correlates edges and causes replication on half the cells [S1].
*   **Pre-shading Limitation:** Patterns are pre-shaded, so they cannot be rotated at render time [S1].

## Operating characteristics
*   **Time Complexity:** Constant time per sample, assuming efficient tile lookup. The shearing and wrapping are pre-computed.
*   **Memory Complexity:** Proportional to the number of tiles (8 or 16) and their resolution.
*   **Strengths:** Eliminates visible repetition in large terrains/volumes without requiring complex procedural noise at render time. Compatible with hardware linear filtering if built correctly.
*   **Limitations:** Requires pre-shading, limiting dynamic lighting. Manual placement of objects in tiles is required to avoid artifacts. Limited variability compared to fully procedural methods.
*   **When to Use:** Real-time rendering of large, static or semi-static scenes (e.g., forests) where repetition is noticeable and pre-shading is acceptable.
*   **When Not to Use:** Dynamic lighting scenarios, or when high variability and organic randomness are required without pre-computation.

## Background
Originated in the context of rendering forest scenes in real-time, specifically to avoid repetitivity when mapping texcells onto a terrain. The method relies on the aperiodic scheme described in [NC99] and is discussed in detail in [S1]. It addresses the limitation of pre-shaded textures, which cannot be rotated, by using separate boundary conditions in each direction.

## Sources
[S1] Rendering Forest Scenes in Real-Time, https://doi.org/10.2312/egwr/egsr04/093-102, fetched 2026-10-26
