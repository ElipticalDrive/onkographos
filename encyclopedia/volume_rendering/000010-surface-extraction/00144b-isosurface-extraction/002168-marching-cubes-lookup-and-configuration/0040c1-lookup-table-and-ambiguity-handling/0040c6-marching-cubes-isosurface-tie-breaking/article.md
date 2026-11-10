# Marching Cubes Isosurface Tie-Breaking

Also called: Marching Cubes Ambiguity Resolution, Marching Cubes Tie-Breaking

## Overview
This method resolves topological inconsistencies in the Marching Cubes algorithm that arise when the isosurface value exactly equals a voxel's scalar value. It ensures that adjacent voxels sharing a face classify that face identically, preventing mesh holes caused by incompatible triangulations. It differs from general ambiguity handling (e.g., [Topology and Ambiguity Handling](6547-topology-and-ambiguity-handling.md)) by focusing specifically on the deterministic resolution of exact ties rather than saddle-point topology correction.

## How it works
The method modifies the voxel classification step before the lookup table index is computed. It applies a deterministic tie-breaking rule to residuals that are exactly zero, ensuring consistent sign assignment across shared boundaries.

**Inputs:**
*   Scalar field values $V_i$ at the 8 vertices of a cube ($i=0..7$).
*   Isosurface value $I$.
*   Machine epsilon $\epsilon_{mach}$ (e.g., `np.spacing(1.0)` for float32/float64).

**Outputs:**
*   Modified residuals $R'_i$ used for classification.
*   Lookup table index $C$ derived from $R'_i$.

**Data Structures:**
*   No new data structures. The method operates on the residuals used to generate the case index $C$ defined in [Lookup Table and Ambiguity Handling](10327-lookup-table-and-ambiguity-handling.md).

**Steps:**
1.  Calculate the residual for each vertex: $R_i = V_i - I$.
2.  Check if the residual is effectively zero. A residual is considered an exact tie if its magnitude is below machine epsilon: $|R_i| < \epsilon_{mach}$.
3.  Apply the tie-break:
    *   If $|R_i| < \epsilon_{mach}$, set $R'_i = \epsilon_{small}$, where $\epsilon_{small}$ is a small positive value (typically $\epsilon_{mach}$ or a fixed small constant like $10^{-6}$). This forces the vertex to be classified as "above" the isosurface.
    *   Otherwise, set $R'_i = R_i$.
4.  Compute the case index $C$ using the standard bit-setting convention based on the sign of $R'_i$ (e.g., if $R'_i \ge 0$, bit $i$ is 1; else 0).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def classify_cube(vertices, isovalue):
    """
    Classify a cube using Marching Cubes tie-breaking.
    
    Args:
        vertices: Array of 8 scalar values at cube corners.
        isovalue: The isosurface threshold.
        
    Returns:
        case_index: Integer 0-255 for lookup table indexing.
    """
    # Step 1: Calculate residuals
    residuals = vertices - isovalue
    
    # Step 2 & 3: Apply tie-breaking
    # Machine epsilon for the dtype of residuals
    eps_mach = np.spacing(residuals.dtype.type(1.0))
    
    # Identify exact ties (magnitude below machine epsilon)
    is_exact_tie = np.abs(residuals) < eps_mach
    
    # Create modified residuals
    # If exact tie, force to small positive value. 
    # Otherwise, keep original residual.
    # Note: The flawed implementation forced ALL values below eps_mach to positive,
    # breaking scale invariance. The correct implementation only affects exact ties.
    modified_residuals = np.where(is_exact_tie, eps_mach, residuals)
    
    # Step 4: Compute case index
    # Standard convention: bit i is 1 if modified_residual[i] >= 0
    case_index = 0
    for i in range(8):
        if modified_residuals[i] >= 0:
            case_index |= (1 << i)
            
    return case_index
```

**Parameters:**
*   $\epsilon_{mach}$: Machine epsilon for the floating-point type used. For float32, $\approx 1.19 \times 10^{-7}$; for float64, $\approx 2.22 \times 10^{-16}$.
*   $\epsilon_{small}$: A small positive value. Often set to $\epsilon_{mach}$ or a fixed small constant. Must be positive to ensure the "above" classification.

**Edge Cases and Pitfalls:**
*   **Scale Invariance:** Applying the tie-break to *nonzero* residuals (where $|V_i - I| < \epsilon_{mach}$ but $V_i \neq I$) breaks scale invariance. If the scalar field is multiplied by a small scale factor, near-isosurface values may be incorrectly forced to positive epsilon, changing the surface topology or causing "No surface found" errors [S1, unofficial]. The tie-break must only apply to *exact* zero residuals (or values indistinguishable from zero due to precision limits in the specific context of exact equality).
*   **Shared Faces:** The primary goal is to ensure that two adjacent voxels sharing a face compute the same classification for the 4 shared vertices. If one voxel classifies a vertex as "below" and the other as "above" due to floating-point rounding differences, the shared face will have inconsistent edge intersections, leading to cracks.
*   **Exact Zero:** The tie-break is only necessary when $V_i = I$ exactly. In most practical scenarios with continuous fields, exact equality is rare, but it occurs in synthetic data or when the isovalue is chosen to match a data value.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel. The tie-breaking check is a constant-time comparison and assignment.
*   **Memory Complexity:** $O(1)$ additional memory per voxel (for the modified residuals).
*   **Strengths:** Eliminates mesh holes caused by exact ties. Deterministic.
*   **Limitations:** Does not resolve topological ambiguities (saddle points) in cases 46, 47, 98, 99, 210, 213, 214, 215. Those require more complex handling (e.g., [Topology and Ambiguity Handling](6547-topology-and-ambiguity-handling.md)).
*   **When to use:** When exact ties between voxel values and isovalue are possible or likely.
*   **When not to use:** When scale invariance is critical and the field values are not normalized, and the implementation incorrectly applies the tie-break to near-zero nonzero residuals.

## Background
The issue of exact ties causing mesh holes was identified in the scikit-image implementation of Marching Cubes. The initial fix applied a tie-break to all values below machine epsilon, which was found to break scale invariance. A corrected implementation restricts the tie-break to actual zero residuals (or values indistinguishable from zero due to precision limits in the context of exact equality) [S1, unofficial]. This method is a specific refinement of the standard Marching Cubes algorithm to handle edge cases in discrete scalar fields.

## Sources
[S1] Fix marching cubes ties at isosurface values, https://github.com/scikit-image/scikit-image/pull/8275, fetched 2026-10-26 (unofficial)
