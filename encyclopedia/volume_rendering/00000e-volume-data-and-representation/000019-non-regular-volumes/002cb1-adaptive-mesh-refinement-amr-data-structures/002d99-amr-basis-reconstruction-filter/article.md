# AMR Basis Reconstruction Filter

Also called: AMR Basis Function Interpolation, Basis Function Reconstruction, Basis function interpolation, Basis interpolation method, Reconstruction via Basis Functions, basis function approach, basis method

## Overview
The AMR Basis Reconstruction Filter is a volume rendering interpolation strategy for block-structured Adaptive Mesh Refinement (AMR) data that computes a continuous, smooth scalar field by blending contributions from all AMR levels simultaneously [S1, unofficial]. It remedies stitching artifacts at level boundaries without relying on non-rectilinear grid interpolation or cell subdivision, distinguishing it from methods that use stitch cells or simplicial elements [S1, unofficial]. Unlike the "Finest" method, which produces high-frequency transition artifacts at coarse boundaries, the Basis method yields smoother, continuous images at the cost of higher computational expense [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A query point $\vec{P}$ in 3D space; the AMR dataset consisting of bricks, each containing cell-centered values $C.v$ and geometric metadata (level, refinement factor $R$, cell width $C.w$).
*   **Output:** A single interpolated scalar value $V(\vec{P})$.

**Data Structures**
The method relies on the Berger-Colella AMR data structure [S1, unofficial].
*   **Brick:** A block of $N \times N \times N$ cells (typically $16^3$).
*   **Cell:** Defined by its center $\vec{C_p}$, its scalar value $C.v$, and its width $C.w$.
*   **Dual Cell:** For a query point $\vec{P}$ at a specific AMR level $l$, the dual cell $D$ is the coarsest cell at that level containing $\vec{P}$. It has 8 corners.
*   **Leaf vs. Inner Node:** A cell is a "leaf" if it is not subdivided further (i.e., it is the finest representation at that location). It is an "inner" node if it has finer children.

**Formulas**
The reconstruction uses hat-shaped basis functions $\hat{H}_C(\vec{P})$ centered at the corners of the dual cells.

The 1D hat function is defined as [S1, unofficial]:
$$ \hat{h}(t) = \max(1 - t, 0) $$

The 3D basis function for a corner cell $C$ at query point $\vec{P}$ is [S1, unofficial]:
$$ \hat{H}_C(\vec{P}) = \hat{h}\left(\frac{|\vec{C_p}_x - \vec{P}_x|}{C.w}\right) \hat{h}\left(\frac{|\vec{C_p}_y - \vec{P}_y|}{C.w}\right) \hat{h}\left(\frac{|\vec{C_p}_z - \vec{P}_z|}{C.w}\right) $$

The final interpolated value is a weighted, renormalized sum [S1, unofficial]:
$$ \text{AMR}_{\text{basis}}(\vec{P}) = \frac{\sum_{C_i} \hat{H}_{C_i}(\vec{P}) C_{i}.v}{\sum_{C_i} \hat{H}_{C_i}(\vec{P})} $$
where the sum iterates over all leaf cells $C_i$ that contribute to the query point.

**Algorithm Steps**
1.  Initialize `sum_weights` = 0 and `sum_weightedValues` = 0.
2.  Iterate through AMR levels $l$ starting from level 0 (coarsest) upwards.
3.  Find the dual cell $D$ for point $\vec{P}$ at level $l$.
4.  For each corner cell $C$ of the dual cell $D$:
    a. Determine if $C$ is a leaf cell.
    b. If $C$ is a leaf cell:
        i. Compute weight $w = \hat{H}_C(\vec{P})$.
        ii. Add $w$ to `sum_weights`.
        iii. Add $w \times C.v$ to `sum_weightedValues`.
5.  Check if any of the corner cells $C$ in the current dual cell $D$ are inner nodes (i.e., have finer children).
    a. If **none** of the corner cells are inner nodes, break the loop (no finer levels will contribute).
    b. If any are inner nodes, continue to the next level $l+1$.
6.  Return `sum_weightedValues` / `sum_weights`.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def AMR_basis(P, amr_data):
    sum_weights = 0.0
    sum_weighted_values = 0.0
    
    # Iterate from coarsest level (0) upwards
    for level in amr_data.levels:
        # Find the dual cell D at this level containing P
        D = find_dual_cell(P, level)
        
        if D is None:
            continue
            
        has_inner_node = False
        level_contribution = False
        
        for corner_cell in D.corners:
            # Check if this corner cell is a leaf (finest available at this location)
            if corner_cell.is_leaf:
                # Calculate basis function weight
                # h(t) = max(1 - t, 0)
                # H_C(P) = h(|Cx - Px|/Cw) * h(|Cy - Py|/Cw) * h(|Cz - Pz|/Cw)
                dx = abs(corner_cell.center.x - P.x) / corner_cell.width
                dy = abs(corner_cell.center.y - P.y) / corner_cell.width
                dz = abs(corner_cell.center.z - P.z) / corner_cell.width
                
                weight = max(1.0 - dx, 0.0) * max(1.0 - dy, 0.0) * max(1.0 - dz, 0.0)
                
                if weight > 0:
                    sum_weights += weight
                    sum_weighted_values += weight * corner_cell.value
                    level_contribution = True
            
            # Check if this corner cell has finer children (is an inner node)
            if corner_cell.has_children:
                has_inner_node = True
        
        # If no corners in this dual cell are inner nodes, 
        # no finer level will contribute to this point.
        if not has_inner_node:
            break
            
    if sum_weights == 0:
        # Fallback if no leaf cells found (e.g., point outside all bricks)
        return 0.0
        
    return sum_weighted_values / sum_weights
```

**Parameters and Edge Cases**
*   **Refinement Factor ($R$):** Typically 2. Cells on finer levels align with boundaries of coarser levels such that one coarse cell is covered by $R \times R \times R$ fine cells [S1, unofficial].
*   **Non-Interpolating Nature:** The method is not strictly interpolating; the basis functions from different levels may overlap and sum to more or less than 1, and the value at a cell center is not guaranteed to equal the cell's stored value if other levels contribute [S1, unofficial].
*   **Coarsest Level Determination:** It is not obvious how to determine the coarsest level that contributes. In deeply refined regions, the algorithm may perform many dual-cell look-ups up to the root level, which is costly [S1, unofficial].
*   **Support Overlap:** The basis function support is not truncated at the first sample on the neighboring side, leading to larger support overlap between cells from neighboring blocks compared to methods like Ljung et al. [S2].

## Subtopics

### AMR Blend Method
A reconstruction strategy for AMR data that renders different AMR levels (e.g., Level 0-1, 0-2) as distinct layers or blended regions, as opposed to the single continuous field produced by the Basis method [S1, unofficial]. It is used in the OSPRay implementation to visualize data at varying levels of refinement [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The cost depends on the depth of refinement. In deeply refined regions, the algorithm may need to traverse from the finest level up to the coarsest level to determine contributions, potentially requiring many dual-cell look-ups [S1, unofficial]. Determining the finest contributing level is easy, but the coarsest is not [S1, unofficial].
*   **Memory Complexity:** Requires efficient access to the AMR hierarchy (e.g., via a kd-tree or multi-octree) to locate dual cells and determine leaf/inner status [S1, unofficial].
*   **Performance:** The Basis method is the most expensive of the four reconstruction strategies provided in the source paper, while "Finest" achieves the highest performance [S1, unofficial].
*   **Quality:** Produces smooth, continuous images without the high-frequency transition artifacts at coarse cell boundaries seen in the "Finest" method [S1, unofficial]. It avoids ghosting artifacts by not using virtual or inner cells [S1, unofficial].
*   **Limitations:** The non-strictly interpolating nature makes implicit ray-isosurface intersection non-obvious [S1, unofficial]. The large support overlap can lead to increased computational load in regions with many neighboring blocks [S2].

## Background
The method was introduced by Ingo Wald, Carson Brownlee, Will Usher, and Aaron Knoll in 2017 as part of a CPU-based volume rendering framework for block-structured (Berger-Colella) AMR data, implemented in OSPRay and ParaView [S1, unofficial]. It was novel compared to existing methods like those by Weber et al. and Moran and Ellsworth, which required subdividing lower-resolution cells into stitch cells or simplicial elements to handle boundaries [S1, unofficial]. The approach borrows concepts from scattered data interpolation techniques, specifically Franke-style interpolation, using hat-shaped basis functions [S1, unofficial].

## Sources
[S1] CPU volume rendering of adaptive mesh refinement data, https://www.researchgate.net/publication/321321949_CPU_volume_rendering_of_adaptive_mesh_refinement_data, fetched 2026-10-26 (unofficial)
[S2] PDF Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
