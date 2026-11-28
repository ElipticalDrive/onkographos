# Vertex Ambient Occlusion Approximation

Also called: Ambient Occlusion (Simple), Voxel Vertex AO, Voxel-based AO approximation

## Overview
Vertex Ambient Occlusion Approximation is a local illumination technique that darkens vertices of a polygon mesh generated from voxel data based on the density of solid voxels in their immediate neighborhood [S2, unofficial]. It solves the problem of harsh, blocky lighting transitions and incorrect zero-light values on blocked faces by sampling neighbors rather than the cell itself [S2, unofficial]. Unlike standard ray-casted ambient occlusion, it achieves a similar darkening effect in concave areas as a side effect of this averaging process, without the computational cost of ray casting [S2, unofficial]. It differs from [Voxel Ambient Occlusion](6646-voxel-ambient-occlusion.md) by operating on mesh vertices derived from the grid rather than shading the voxel cells directly, and from [JAOCC Shading](3634-jaocc-shading.md) by using a simpler neighbor-counting heuristic rather than a precomputed cache or complex occlusion query [G].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel grid representing scene geometry, a mesh vertex $v$ with position $(x, y, z)$ and surface normal $\mathbf{n}$, and the vertex's base color [S2, unofficial].
*   **Output:** An ambient occlusion factor $AO \in [0, 1]$ applied to the vertex color [S2, unofficial].

**Data Structures:**
*   **Voxel Grid:** A 3D array where each cell stores occupancy (boolean or density). Access is bounded; coordinates outside the grid return `false` (non-solid) [S2, unofficial].

**Steps:**
1.  **Initialize Counters:** Set `occluded = 0` and `total = 0` [S2, unofficial].
2.  **Sample Neighbors:** Iterate through all 26 immediate neighbors of the vertex's voxel coordinates $(x, y, z)$. Let the neighbor offsets be $(dx, dy, dz)$ where $dx, dy, dz \in \{-1, 0, 1\}$, excluding $(0, 0, 0)$ [S2, unofficial].
3.  **Directional Check:** For each neighbor, calculate the direction vector $\mathbf{d} = (dx, dy, dz)$. Compute the dot product $dot = \mathbf{d} \cdot \mathbf{n}$ [S2, unofficial].
    *   If $dot < 0$, the neighbor is on the "back" side of the surface relative to the normal; skip this neighbor [S2, unofficial].
    *   If $dot \ge 0$, increment `total` [S2, unofficial].
4.  **Occupancy Check:** If the neighbor voxel at $(x+dx, y+dy, z+dz)$ is solid, increment `occluded` [S2, unofficial].
5.  **Calculate Factor:** Compute the AO factor using the formula:
    $$ AO = 1.0 - \frac{occluded}{total} \times 0.5 $$
    If `total` is 0, set $AO = 1.0$ [S2, unofficial].
    *Note: An alternative formula found in some implementations uses a weighted sum of side and corner neighbors:*
    $$ ao = \frac{side.x + side.y + \max(corner, side.x \times side.y)}{10.0} $$
    where `side` and `corner` are scalar voxel occupancy values (0.0 or 1.0) [S1, unofficial]. The sources do not specify which is the canonical "Vertex Ambient Occlusion Approximation" vs a variant, but the neighbor-counting method is the primary definition in the family context [G].
6.  **Apply Shading:** Multiply the vertex's RGB color components by $AO$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_vertex_ao(vertex_pos, vertex_normal, voxel_grid):
    """
    Computes Ambient Occlusion for a vertex based on voxel neighbors.
    
    Args:
        vertex_pos: Tuple (x, y, z) of integer voxel coordinates.
        vertex_normal: Normalized vector with components x, y, z.
        voxel_grid: Object with method is_solid(x, y, z) returning bool.
                    Returns False for out-of-bounds coordinates.
                    
    Returns:
        float: AO factor in [0.0, 1.0].
    """
    occluded = 0
    total = 0
    
    # Iterate over 26 neighbors (excluding center 0,0,0)
    for dx in [-1, 0, 1]:
        for dy in [-1, 0, 1]:
            for dz in [-1, 0, 1]:
                if dx == 0 and dy == 0 and dz == 0:
                    continue
                
                # Calculate direction vector to neighbor
                dot = dx * vertex_normal.x + dy * vertex_normal.y + dz * vertex_normal.z
                
                # Skip neighbors on the back side of the surface
                if dot < 0:
                    continue
                
                total += 1
                
                # Check if neighbor is solid
                if voxel_grid.is_solid(vertex_pos.x + dx, vertex_pos.y + dy, vertex_pos.z + dz):
                    occluded += 1
    
    # Calculate AO factor
    if total == 0:
        return 1.0
    
    # Formula from [S2, unofficial]
    ao_factor = 1.0 - (occluded / total) * 0.5
    
    return ao_factor
```

**Parameters:**
*   **Neighborhood Radius:** Fixed to 1 voxel (26 neighbors) [S2, unofficial].
*   **Occlusion Weight:** The factor $0.5$ in the formula scales the maximum occlusion effect [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Checks:** Coordinates outside the voxel grid must return `false` (non-solid) to prevent out-of-bounds access errors [S2, unofficial].
*   **Zero Total:** If no neighbors are in the front hemisphere (`total == 0`), the AO factor defaults to 1.0 (no occlusion) [S2, unofficial].

## Subtopics

### [Smooth Vertex Lighting with Ambient Occlusion Effect](10956-smooth-vertex-lighting-with-ambient-occlusion-effect.md)
Calculates lighting for mesh vertices by averaging intensities of four surrounding voxel cells along the face normal. It sets itself apart by producing smoother gradients than the base method's binary neighbor counting, avoiding harsh blocky transitions. It is chosen when continuous lighting is preferred over immediate crevice darkening. See [Smooth Vertex Lighting with Ambient Occlusion Effect](10956-smooth-vertex-lighting-with-ambient-occlusion-effect.md).

### Least-Square Baking
Uses signal processing to minimize interpolation error between uniform mesh samples and derived vertex values [S3, unofficial]. It sets itself apart by optimizing the sampling-reconstruction pipeline rather than direct neighbor counting [S3, unofficial]. It is chosen when high-quality precomputed AO is needed for smooth interpolation across large meshes, at the cost of a more complex baking process [S3, unofficial].

### [Blocky Ambient Occlusion Derivation](17649-blocky-ambient-occlusion-derivation.md)
Computes expected vertex AO purely from local geometry to replace hand-written offset tables. It sets itself apart by using a deterministic derivation based on face normal and vertex position rather than static lookup tables. It is chosen for its robustness against meshing artifacts and offset table errors in blocky voxel engines.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Simple to implement, no ray casting required, provides immediate visual feedback for crevices and concavities [S2, unofficial].
*   **Limitations:** Approximate; the occlusion effect is limited to the immediate 1-voxel neighborhood [S2, unofficial].
*   **When to Use:** Real-time rendering of voxel-based scenes where performance is critical and high-fidelity global illumination is not required [G].
*   **When Not to Use:** When smooth, continuous lighting gradients are essential (use Smooth Vertex Lighting or Least-Square Baking), or when long-range occlusion is needed [G].

## Background
The method originates from the need to add shading detail to voxel-based meshes without the overhead of ray tracing. It is part of the family of voxel-based neighbor counting techniques. Variations and improvements, such as Least-Square Baking, have been explored in academic literature on real-time voxel global illumination [S3, unofficial]. The "Blocky AO" derivation was developed to fix inconsistencies in meshing pipelines, specifically replacing hand-tuned offset tables with a geometric derivation [S1, unofficial].

## Sources
[S1] ria8651/bevy-voxel-engine, https://github.com/ria8651/bevy-voxel-engine, fetched 2026-10-27 (unofficial)
[S2] elyssov/prometheus-engine, https://github.com/elyssov/prometheus-engine, fetched 2026-10-28 (unofficial)
[S3] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
