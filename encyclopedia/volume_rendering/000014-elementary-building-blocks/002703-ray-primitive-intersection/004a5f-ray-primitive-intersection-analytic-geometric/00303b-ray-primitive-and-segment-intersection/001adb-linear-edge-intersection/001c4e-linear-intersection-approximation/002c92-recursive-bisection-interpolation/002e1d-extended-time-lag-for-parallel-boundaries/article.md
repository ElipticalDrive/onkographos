# Extended Time Lag for Parallel Boundaries

Also called: Border time lag, Prevention sphere, Time Lag Extension for Block Borders

## Overview
Extended Time Lag for Parallel Boundaries is a geometric constraint technique used during hierarchical isosurface extraction to prevent refinement dependencies and topological artifacts at the interfaces of adjacent sub-blocks [S1, S2]. It extends the standard time lag method by enforcing a local inclusion check against block boundaries, ensuring that edge contractions do not create geometric inconsistencies between neighbors [S1]. Unlike the original time lag which applies globally, this extension specifically targets the borders of current sub-blocks to maintain consistency in parallel or partitioned rendering environments [S1].

## How it works
The method operates during the edge collapse phase of simplification algorithms (such as the tandem algorithm) within a hierarchical block structure. It prevents the contraction of an edge if the resulting geometric influence sphere would extend outside the current block's boundaries.

**Inputs and Outputs:**
*   **Input:** An edge $ab$ to be considered for collapse, the current block boundary definition, and the radii of the edge endpoints $rad(a)$ and $rad(b)$.
*   **Output:** A boolean decision indicating whether the edge contraction is permitted or prevented.

**Data Structures:**
*   **Edge:** Defined by endpoints $a$ and $b$.
*   **Block:** Defined by its spatial boundaries (e.g., axis-aligned bounding box).
*   **Radius Field:** Each vertex $x$ stores a scalar value $rad(x)$.

**Formulas:**
The radius of a point $c$, which is the result of collapsing edge $ab$ into a single vertex, is calculated as [S1, S2]:

$$
rad(c) = \frac{\|a - b\| + rad(a) + rad(b)}{2}
$$

Where:
*   $\|a - b\|$ is the Euclidean distance between vertices $a$ and $b$.
*   $rad(a)$ and $rad(b)$ are the radii associated with vertices $a$ and $b$.
*   For original vertices of the isosurface, the radius is defined as [S1, S2]:
    $$
    rad(x) = 1
    $$

**Algorithm Steps:**
1.  Identify the edge $ab$ to be collapsed.
2.  Calculate the midpoint $m$ of the edge: $m = \frac{a + b}{2}$.
3.  Calculate the radius $rad(c)$ of the potential new vertex $c$ using the formula above.
4.  Define a sphere $S$ centered at $m$ with radius $rad(c)$.
5.  Check if sphere $S$ is **totally included** within the current block's boundaries.
    *   If $S$ is not totally included (i.e., it intersects or crosses the block boundary), **prevent** the contraction of edge $ab$ [S1, S2].
    *   If $S$ is totally included, allow the contraction [S1, G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def is_edge_contraction_allowed(edge_ab, block_boundary, rad_map):
    """
    Determines if edge ab can be collapsed based on Extended Time Lag constraints.
    
    Args:
        edge_ab: Tuple (a, b) where a and b are vertex objects with position attributes.
        block_boundary: Object defining the spatial limits of the current block.
        rad_map: Dictionary mapping vertex ID to radius value.
        
    Returns:
        bool: True if contraction is allowed, False if prevented.
    """
    a, b = edge_ab
    
    # 1. Calculate radius of the resulting vertex c
    # rad(c) = (||a - b|| + rad(a) + rad(b)) / 2
    dist_ab = vector_length(subtract(a.position, b.position))
    rad_a = rad_map.get(a.id, 1) # Default to 1 if not found, though original vertices should be present
    rad_b = rad_map.get(b.id, 1)
    rad_c = (dist_ab + rad_a + rad_b) / 2
    
    # 2. Define the sphere
    # Center is the midpoint of ab
    center = add(a.position, multiply_scalar(subtract(b.position, a.position), 0.5))
    radius = rad_c
    
    # 3. Check inclusion in block
    # The sphere is totally included if the distance from the center to any face
    # of the block is greater than or equal to the radius.
    # Assuming block_boundary has methods to get min/max extents or distance to boundary.
    if is_sphere_inside_block(center, radius, block_boundary):
        return True
    else:
        return False

def is_sphere_inside_block(center, radius, block_boundary):
    """
    Checks if a sphere is completely contained within an axis-aligned bounding box.
    """
    # Get block extents
    min_pt = block_boundary.min
    max_pt = block_boundary.max
    
    # Check if the sphere extends beyond any boundary
    # Sphere extends beyond min_x if center.x - radius < min_pt.x
    if (center.x - radius < min_pt.x or 
        center.x + radius > max_pt.x or
        center.y - radius < min_pt.y or 
        center.y + radius > max_pt.y or
        center.z - radius < min_pt.z or 
        center.z + radius > max_pt.z):
        return False
        
    return True
```

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point errors in distance calculations may cause a sphere that is mathematically inside to be flagged as outside, or vice versa. A small epsilon tolerance may be required in the inclusion check [G].
*   **Original Vertices:** The radius definition $rad(x)=1$ applies strictly to original vertices. If intermediate vertices are created, their radii must be propagated correctly using the formula to maintain the "prevention sphere" logic [S1, S2].
*   **Boundary Definition:** The block boundary must be clearly defined relative to the coordinate system of the vertices. In hierarchical splitting, the boundary of a child block is a subset of the parent, and the check must use the specific child block's limits [S1].

## Subtopics

### Recursive Bisection Interpolation
The parent method that refines edge intersection points by recursively bisecting edges. Extended Time Lag is applied during the simplification phase that follows or accompanies this refinement to ensure topological consistency across LOD boundaries [Parent](5298-recursive-bisection-interpolation.md).

## Operating characteristics
*   **Time Complexity:** The check adds a constant-time geometric test per edge considered for collapse. It does not change the asymptotic complexity of the simplification algorithm but adds a small constant overhead per edge [S1, S2].
*   **Memory Complexity:** No additional significant memory is required beyond storing the block boundaries and the radius field for vertices, which is already part of the tandem algorithm's data structure [S1, S2].
*   **Strengths:** Effectively prevents "refinement dependencies" and "artifacts due to bias in shape during simplification" at block borders [S1, S2]. It ensures that simplified surfaces in adjacent blocks do not exhibit gaps or overlaps caused by independent simplification decisions [S1].
*   **Limitations:** It may prevent valid simplifications near boundaries, potentially leaving more vertices near block edges than would be necessary in a global simplification. This can lead to slightly higher polygon counts at interfaces [S1].
*   **When to use:** Use when extracting isosurfaces from large datasets partitioned into blocks (e.g., for parallel processing or hierarchical LOD) where visual continuity across block boundaries is critical [S1, S2].

## Background
The method was introduced in the context of isosurface extraction and interpretation for very large geophysical datasets [S1]. It extends the "time lag" concept, originally designed to prevent refinement dependencies globally, to specifically handle the boundaries of sub-blocks in a parallel or hierarchical extraction framework [S1, S2]. The technique is particularly relevant for algorithms like the tandem algorithm, which is designed for large-scale data but requires careful handling of block interfaces to maintain geometric quality [S2].

## Sources
[S1] Isosurface extraction and interpretation on very large datasets in geophysics, https://inria.hal.science/inria-00277070v1, fetched 2026-10-26
[S2] Parallel extraction and simplification of large isosurfaces using an extended tandem algorithm, https://doi.org/10.1016/j.cad.2009.04.016, fetched 2026-10-26
