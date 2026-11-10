# Coarsest-Level Lerp

Also called: Coarsest-Level Interpolation, Coarsest-Level Trilinear Interpolation

## Overview
Coarsest-Level Lerp is a method for computing the value of a coarser-side octant vertex in Adaptive Mesh Refinement (AMR) volume rendering by performing trilinear interpolation on the coarsest level involved for any of the inputs [S1, unofficial]. It solves the problem of generating consistent, continuous interpolants at level boundaries where a coarse octant meets finer children [S1, unofficial]. It sets itself apart from [Finest Level Lerp](2945-finest-level-lerp.md) by utilizing the coarsest available resolution rather than the finest, and from [Blending Between Levels (Fast)](7871-blending-between-levels-fast.md) by being a deterministic, non-blending average that avoids the computational overhead of transparency-based blending [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The current octant `oct` (with level `oct.l` and position `oct[X'].p`), and the dual cell `D` associated with the vertex being computed. `D` contains level information `D[0'].l` and `D[X'].l` for the adjacent cells.
*   **Output:** The interpolated vertex value `oct[X'].v`.

**Data Structures:**
*   `oct`: Represents an octant in the AMR hierarchy.
    *   `oct.l`: Integer level index of the octant.
    *   `oct[X'].p`: Position vector of the vertex.
    *   `oct[X'].v`: Scalar value to be computed.
*   `D`: Dual cell structure representing the interface between cells.
    *   `D[0'].l`: Level index of the reference cell.
    *   `D[X'].l`: Level index of the neighbor cell.
    *   `D.lerp(P)`: Function performing trilinear interpolation at position `P` using the dual cell's data.

**Algorithm Steps:**
1.  **Calculate Logical Coordinate Index:** Determine the minimum level involved in the interpolation by comparing the levels of the reference cell and the neighbor cell within the dual cell structure:
    $$ lX' = \min(D[0'].l, D[X'].l) $$
2.  **Check Level Equality:** Compare the calculated minimum level $lX'$ with the current octant's level `oct.l`.
3.  **Branching Logic:**
    *   **Case A (Same Level):** If $lX' == \text{oct}.l$, the vertex lies on a boundary where both sides are at the same resolution. Compute the value as the arithmetic average of the input values from the dual cell components:
        $$ \text{oct}[X'].v = \text{avg}(D[0'].v, D[X'].v) $$
    *   **Case B (Different Levels):** If $lX' \neq \text{oct}.l$, perform trilinear interpolation on the coarsest level $lX'$. Call the helper function `lerpOnLevel` with the calculated level and the vertex position:
        $$ \text{oct}[X'].v = \text{lerpOnLevel}(lX', \text{oct}[X'].p) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_coarser_side_vertex(oct, D, X_prime):
    """
    Computes the value for a coarser-side octant vertex using Coarsest-Level Lerp.
    
    Args:
        oct: The current octant object.
        D: The dual cell structure containing level and value info for adjacent cells.
        X_prime: Index/identifier for the vertex on the octant face.
        
    Returns:
        The interpolated vertex value.
    """
    # Step 1: Calculate logical coordinate index (minimum level involved)
    lX_prime = min(D[0].l, D[X_prime].l)
    
    # Step 2: Check if the minimum level matches the current octant's level
    if lX_prime == oct.l:
        # Step 3a: Same level - average the inputs
        # Note: D[0].v and D[X_prime].v are the scalar values from the dual cell components
        oct[X_prime].v = (D[0].v + D[X_prime].v) / 2.0
    else:
        # Step 3b: Different levels - interpolate on the coarsest level
        # lerpOnLevel performs trilinear interpolation on the structured grid at level lX_prime
        oct[X_prime].v = lerp_on_level(lX_prime, oct[X_prime].p)
        
    return oct[X_prime].v

def lerp_on_level(level, position):
    """
    Performs trilinear interpolation on a specific AMR level.
    Views the level as a structured grid.
    """
    # Find the dual cell for the given position at the specified level
    dual_cell = find_dual_cell(level, position)
    # Perform trilinear interpolation using the dual cell
    return dual_cell.lerp(position)

def find_dual_cell(level, position):
    """
    Finds the dual cell D at the given logical level containing the position.
    Implementation detail: Maps position to grid coordinates at 'level' resolution.
    """
    # Not given in the sources. Standard AMR grid lookup.
    pass
```

**Parameters and Typical Values:**
*   **Level Selection:** The method strictly uses the *minimum* of the involved levels. No tunable parameters are required for the core logic.
*   **Optimization:** In most cases, multiple `lerpOnLevel` calls find the same dual cell `D`. This can be detected and replaced with direct averaging for performance [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on the stability of `lerpOnLevel`. If the coarsest level is significantly coarser than the finest, interpolation error may increase.
*   **Boundaries:** The method assumes the existence of a valid dual cell `D` at the coarsest level. If the coarsest level does not cover the position, the method fails unless a global coarsest level is defined.
*   **Degenerate Input:** If `D[0].l` or `D[X'].l` are undefined (e.g., outside the domain), the method is undefined.

## Subtopics

### Current-Level Lerp
A method that performs interpolation on the leaf cell (current level) rather than the coarsest level, making the interpolant adaptive [S1, unofficial]. It calculates the logical coordinate index $lX' = \min(D[0'].l, D[X'].l)$ and checks if $lX' == \text{oct}.l$. If true, it averages the inputs. Otherwise, it finds the leaf cell level using `findLeafCell(oct[X'].p).l` and interpolates on that level: $\text{oct}[X'].v = \text{lerpOnLevel}(\text{level}, \text{oct}[X'].p)$ [S1, unofficial]. It sets itself apart from Coarsest-Level Lerp by adapting to the finest available resolution at the sample point, offering higher accuracy in refined regions but potentially discontinuous behavior across level boundaries [S2, unofficial].

### Finest-Level Interpolation
Trilinear interpolation performed on a fixed logical level $l$. It views the level as a structured grid where non-existing cells are defined via the Closest Existing Cell (CEC) operator [S2, unofficial]. It is cheap and continuous but not adaptive [S2, unofficial]. Unlike Coarsest-Level Lerp, which dynamically selects the coarsest level involved in the boundary, Finest-Level Interpolation uses a pre-selected fixed level, ignoring the local refinement hierarchy for the interpolation step [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per vertex, assuming constant-time lookup for `findDualCell` and `lerpOnLevel`. The cost is dominated by the trilinear interpolation on the coarsest level.
*   **Memory Complexity:** $O(1)$ auxiliary memory. No additional data structures are required beyond the existing AMR grid and dual cell structures.
*   **Measured Performance:** Not given in the sources. However, the source notes that detecting repeated dual cell lookups and replacing them with direct averaging improves performance [S1, unofficial].
*   **Strengths:** Guarantees continuity and accuracy by using the coarsest level involved, ensuring that the interpolant does not introduce artifacts from overly fine resolutions at coarse boundaries [S1, unofficial]. It is simple to implement given a `lerpOnLevel` interface.
*   **Limitations:** May be less accurate than adaptive methods (like Current-Level Lerp) in regions with high refinement, as it ignores the finer details available at the vertex location [S1, unofficial]. It is not adaptive [S2, unofficial].
*   **When to Use:** When continuity and consistency across level boundaries are prioritized over local accuracy, or when the coarsest level provides sufficient resolution for the visual output.
*   **When Not to Use:** When high-frequency details in refined regions are critical and the coarsest level is too coarse to capture them.

## Background
Coarsest-Level Lerp was introduced in the context of CPU isosurface ray tracing of Adaptive Mesh Refinement data [S1, unofficial]. It is one of four options introduced to guarantee that the interpolant is continuous, adaptive, accurate, and locally rectilinear [S1, unofficial]. The method builds on the logical grid abstraction of AMR data, where each refinement level is viewed as a structured grid [S1, unofficial]. It is designed to work with the `lerpOnLevel` interface, which provides trilinear interpolation based on the cell [S1, unofficial].

## Sources
[S1] PDF CPU Isosurface Ray Tracing of Adaptive Mesh Refinement Da, https://www.sci.utah.edu/~feng/assets/papers/Fen-AMR-VIS18.pdf, fetched 2026-10-26 (unofficial)
[S2] CPU volume rendering of adaptive mesh refinement data, https://www.researchgate.net/publication/321321949_CPU_volume_rendering_of_adaptive_mesh_refinement_data, fetched 2026-10-26 (unofficial)
