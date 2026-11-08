# Mod2 Reduction

Also called: Modulo-2 Reduction, Subgrid Mod 2 Algorithm

## Overview
Mod2 Reduction is an ablation of the Subgrid Marching Tetrahedra method that provides exact input intersection locations but limits the output to at most one intersection per edge [S1]. It uses the parity of the intersection count to determine connectivity, effectively restoring the single-surface-per-edge constraint of classic Marching Tetrahedra while utilizing subgrid-accurate vertex positions [S1]. It is distinguished from its sibling [Subgrid Surface Reconstruction](1666-subgrid-surface-reconstruction.md) by its strict limitation to one output vertex per edge, whereas the full subgrid method allows multiple vertices to represent complex topology [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** For each edge of a tetrahedron, a list of intersection parameter values ($t$) where the scalar field crosses the isovalue, and the total count of these intersections.
*   **Output:** A single aggregated intersection location (vertex position) for the edge, or a flag indicating the edge should be skipped in the current tetrahedron's processing.

**Data Structures:**
*   **Edge Intersections:** A list of floating-point values representing the parametric position $t \in [0, 1]$ along the edge where the isosurface is crossed.
*   **Intersection Count:** An integer representing the size of the list above.

**Steps:**
1.  **Parity Check:** Calculate the parity of the intersection count $N$ for the edge: $p = N \pmod 2$.
2.  **Connectivity Determination:**
    *   If $p = 1$, the two endpoints of the edge are on different sides of the surface. The edge intersects the surface an odd number of times.
    *   If $p = 0$, the endpoints are on the same side (or the surface does not cross the edge net). The edge does not contribute to the mesh connectivity for this tetrahedron [S1].
3.  **Aggregation:** If $p = 1$, calculate the aggregated intersection location $V_{agg}$ by averaging all true intersection positions $t_i$ found along the edge:
    $$ V_{agg} = \frac{1}{N} \sum_{i=1}^{N} t_i $$
    This position is then used as the primal vertex for the edge in the mesh construction [S1].
4.  **Pipeline Integration:** In the dual subgrid pipeline, this function is called if the `opts.mod2` flag is true. If the parity check indicates no net intersection ($p=0$), the current tetrahedron is skipped [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mod2_reduction(edge_intersections: list[float], edge_count: int) -> tuple[bool, float]:
    """
    Determines connectivity and aggregated position for an edge.
    
    Args:
        edge_intersections: List of t-values where the isosurface crosses the edge.
        edge_count: Number of intersections (len(edge_intersections)).
        
    Returns:
        (should_skip, aggregated_t)
        should_skip: True if the edge does not cross the surface (even parity).
        aggregated_t: The average t-value if odd parity, otherwise undefined.
    """
    # Step 1 & 2: Parity check
    if edge_count % 2 == 0:
        # Even number of intersections: endpoints are on the same side.
        # Skip this edge for connectivity.
        return (True, 0.0)
    
    # Step 3: Aggregation
    # Odd number of intersections: endpoints are on different sides.
    # Calculate average of all intersection positions.
    sum_t = 0.0
    for t in edge_intersections:
        sum_t += t
    
    avg_t = sum_t / edge_count
    
    # Return False to indicate the edge IS active (should not skip)
    return (False, avg_t)
```

**Parameters:**
*   `opts.mod2`: A boolean flag in the pipeline configuration. When true, Mod2 Reduction is applied instead of the full subgrid intersection handling [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Averaging multiple $t$ values assumes the intersections are distinct and well-separated. If intersections are extremely close, floating-point error may affect the average [G].
*   **Degenerate Input:** If `edge_count` is 0, the parity is even, and the edge is skipped. This is consistent with the definition [S1].
*   **Implementation Note:** The function `convert_mod2_intersections` was moved to `query/mod2_reduction.h` in the reference implementation [S2, unofficial].

## Subtopics

### Subgrid Surface Reconstruction
Allows multiple output vertices per edge to represent complex topology, unlike Mod2 which is limited to one. See [Subgrid Surface Reconstruction](1666-subgrid-surface-reconstruction.md).

## Operating characteristics
*   **Time Complexity:** Linear in the number of intersections per edge. The aggregation step requires a single pass over the intersection list.
*   **Memory Complexity:** Minimal overhead; requires storing the list of intersections temporarily for averaging.
*   **Performance:** In a $100^3$ grid comparison, Subgrid Mod 2 shows slightly better geometric quality than classic Marching Tetrahedra but cannot produce more than one disc per tetrahedron [S1].
*   **Strengths:** Improves the accuracy of primal vertex positions compared to classic Marching Tetrahedra by using exact intersection locations [S1].
*   **Limitations:** Produces much more distortion than the full Subgrid Marching Tetrahedra algorithm because it cannot handle multiple intersections per edge [S1]. It produces the same mesh connectivity as classic Marching Tetrahedra [S1].
*   **When to use:** When the surface topology is simple (no multiple intersections per edge) but higher vertex accuracy than classic Marching Tetrahedra is desired.
*   **When not to use:** When the surface has complex topology with multiple intersections per edge, as this method will fail to represent them correctly [S1].

## Background
The Subgrid Mod 2 Algorithm was introduced as an ablation study to determine the essential components of the Subgrid Marching Tetrahedra method [S1]. It was proposed by the authors of the Subgrid Marching Tetrahedra paper to isolate the contribution of exact intersection locations versus the ability to handle multiple intersections per edge [S1]. The implementation is available in the `hbaktash/subgrid-marching` repository [S2, unofficial].

## Sources
[S1] Subgrid Marching Tetrahedra, https://doi.org/10.1145/3811358, fetched 2026-10-26
[S2] hbaktash/subgrid-marching, https://github.com/hbaktash/subgrid-marching, fetched 2026-10-27 (unofficial)
