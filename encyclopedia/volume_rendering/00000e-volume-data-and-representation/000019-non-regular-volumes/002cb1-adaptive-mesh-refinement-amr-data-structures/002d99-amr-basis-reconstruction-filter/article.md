# AMR Basis Reconstruction Filter

Also called: AMR Basis Function Interpolation, Basis Function Reconstruction, Basis function interpolation, Basis interpolation method, Reconstruction via Basis Functions, basis function approach, basis method

## Overview
The AMR Basis Reconstruction Filter is a volume rendering interpolation strategy for Berger-Colella (BC) Adaptive Mesh Refinement (AMR) data that computes a continuous scalar field by blending contributions from all AMR levels simultaneously [S1, unofficial]. It remedies stitching artifacts at resolution boundaries without subdividing coarse cells into stitch elements or relying on non-rectilinear grid interpolation [S1, unofficial]. Unlike the "Finest" method, which produces high-frequency transition artifacts at coarse cell boundaries, the Basis method yields smooth, continuous images by using a weighted, renormalized sum of hat-shaped basis functions centered at dual cell corners [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A query point $\vec{P}$ in 3D space; the AMR dataset consisting of bricks $B^{(i)}$ at various levels $L^{(i)}$, where each brick contains cell-centered scalar values $C.v$.
*   **Output:** A single interpolated scalar value $V(\vec{P})$.

**Data Structures:**
The method relies on the spatial hierarchy of the AMR data.
*   **Bricks:** Blocks of typically $16 \times 16 \times 16$ cells [S1, unofficial].
*   **Levels:** Hierarchical refinement levels. Finer bricks align with cell boundaries of coarser levels, such that each coarser cell is covered by exactly $R \times R \times R$ finer cells, where $R$ is the refinement factor [S1, unofficial].
*   **Dual Cell $D$:** For a query point $\vec{P}$ at a specific level $l$, the dual cell is the coarsest cell at that level containing $\vec{P}$. Its corners are defined by the vertices of the cell enclosing $\vec{P}$ [S1, unofficial].
*   **Leaf vs. Inner Nodes:** Cells are classified as "leaf" (finest resolution present at that location) or "inner" (coarser cells that are subdivided by finer levels) [S1, unofficial].

**Formulas:**
The reconstruction uses hat-shaped basis functions $\hat{H}_C(\vec{P})$ centered at the corners of the dual cell.

The 1D hat function is defined as:
$$ \hat{h}(t) = \max(1 - t, 0) $$

The 3D basis function for a corner cell $C$ at query point $\vec{P}$ is:
$$ \hat{H}_C(\vec{P}) = \hat{h}\left(\frac{|\vec{C}_p.x - \vec{P}_x|}{\vec{C}_w}\right) \hat{h}\left(\frac{|\vec{C}_p.y - \vec{P}_y|}{\vec{C}_w}\right) \hat{h}\left(\frac{|\vec{C}_p.z - \vec{P}_z|}{\vec{C}_w}\right) $$
where $\vec{C}_p$ is the position of the corner and $\vec{C}_w$ is the support width (half-width) of the cell [S1, unofficial].

The final interpolated value is the weighted sum normalized by the total weight:
$$ \text{AMR}_{\text{basis}}(\vec{P}) = \frac{\sum_{C_i} \hat{H}_{C_i}(\vec{P}) C_{v_i}}{\sum_{C_i} \hat{H}_{C_i}(\vec{P})} $$
where the sum iterates over all leaf cells $C_i$ that contribute to the query [S1, unofficial].

**Algorithm Steps:**
1.  Initialize `sum_weights` = 0 and `sum_weightedValues` = 0.
2.  Iterate through levels $l$ starting from the coarsest level (level 0) upwards.
    *   Find the dual cell $D$ for point $\vec{P}$ at level $l$.
    *   For each corner cell $C$ of the dual cell $D$:
        *   If $C$ is a leaf cell:
            *   Compute $\hat{H}_C(\vec{P})$.
            *   `sum_weights` += $\hat{H}_C(\vec{P})$
            *   `sum_weightedValues` += $\hat{H}_C(\vec{P}) \times C.v$
        *   Check if any corner cell $C$ in $D$ is an inner node.
            *   If **none** of the corner cells in $D$ are inner nodes, break the loop (no finer levels will contribute to this location) [S1, unofficial].
3.  Return `sum_weightedValues` / `sum_weights` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def AMR_basis(P, amr_data):
    sum_weights = 0.0
    sum_weighted_values = 0.0
    
    # Iterate from coarsest level (0) to finest
    for level in amr_data.levels:
        # Find the dual cell D for point P at the current level
        D = find_dual_cell(P, level, amr_data)
        
        # Check if this level has any finer children (inner nodes)
        has_inner_nodes = False
        for corner_cell in D.corners:
            if corner_cell.is_inner:
                has_inner_nodes = True
                break
        
        # Process corner cells
        for corner_cell in D.corners:
            if corner_cell.is_leaf:
                # Calculate basis function value
                # C_p is corner position, C_w is cell half-width
                dx = abs(corner_cell.pos.x - P.x) / corner_cell.half_width
                dy = abs(corner_cell.pos.y - P.y) / corner_cell.half_width
                dz = abs(corner_cell.pos.z - P.z) / corner_cell.half_width
                
                h_x = max(1.0 - dx, 0.0)
                h_y = max(1.0 - dy, 0.0)
                h_z = max(1.0 - dz, 0.0)
                
                H_hat = h_x * h_y * h_z
                
                sum_weights += H_hat
                sum_weighted_values += H_hat * corner_cell.value
        
        # If no inner nodes, we are at the finest contributing level
        if not has_inner_nodes:
            break
            
    if sum_weights == 0:
        return 0.0 # Or handle error appropriately
        
    return sum_weighted_values / sum_weights
```

**Parameters and Edge Cases:**
*   **Refinement Factor $R$:** Typically determines the grid alignment. The method assumes finer bricks align with coarser cell boundaries [S1, unofficial].
*   **Non-Interpolating Nature:** The method is not strictly interpolating; values at cell centers are not guaranteed to equal the stored cell values because basis functions from different levels overlap and do not sum to 1 in a partition-of-unity sense across all levels simultaneously [S1, unofficial].
*   **Ray-Isosurface Intersection:** Because the field is not strictly interpolating, implicit ray-isosurface intersection (e.g., Newton-Raphson) is non-trivial and may not converge reliably [S1, unofficial].
*   **Deep Refinement Cost:** In deeply refined regions, determining the coarsest level that contributes is not obvious. The algorithm may perform many dual-cell look-ups up to the coarsest level before finding the first leaf contributions, which is costly [S1, unofficial].

## Subtopics

### AMR Blend Method
A reconstruction strategy for AMR data that renders different AMR levels (e.g., Level 0-1, 0-2) by blending them, as opposed to the basis method which blends across all levels simultaneously. It is used to visualize specific subsets of the hierarchy. It is implemented in OSPRay for rendering AMR data at varying levels [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** The Basis method is the most expensive of the four reconstruction strategies provided in the source implementation [S1, unofficial]. In deeply refined regions, performance degrades because the algorithm may need to traverse many levels to find the coarsest contributing level, requiring numerous dual-cell look-ups [S1, unofficial].
*   **Memory:** Requires efficient access to the AMR hierarchy (e.g., via a kd-tree) to locate dual cells and determine leaf/inner status [S1, unofficial].

**Measured Performance:**
*   In benchmarks on a KNL node and dual-Xeon systems, the Basis method produced the highest quality images but had the lowest performance compared to "Finest" and other strategies [S1, unofficial].
*   Performance is largely oblivious to the maximum refinement level if the majority of waves are at coarser levels, as finer levels affect only a small portion of rays [S1, unofficial].
*   The SPMD implementation on many-core architectures suffered from code- and data-divergence in the reconstruction kernels [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Smooth, continuous interpolation; avoids ghosting artifacts associated with virtual/inner cells; does not require subdividing cells into stitch elements or simplicial meshes [S1, unofficial].
*   **Limitations:** Not strictly interpolating; difficult to use for implicit ray-isosurface intersection; computationally expensive in deep refinement due to level traversal overhead; larger support overlap between cells from neighboring blocks compared to truncated methods [S1, unofficial][S2].

**When to Use:**
Use when image quality and continuity across AMR boundaries are prioritized over performance, and when implicit ray marching is not required. Suitable for CPU-based interactive ray tracing where smoothness is critical [S1, unofficial].

## Background
The AMR Basis Reconstruction Filter was introduced by Ingo Wald, Carson Brownlee, Will Usher, and Aaron Knoll in 2017 as part of a general method for CPU volume rendering of Berger-Colella AMR data using a kd-tree structure [S1, unofficial]. It was implemented in the OSPRay framework and integrated into ParaView [S1, unofficial]. The method draws inspiration from scattered data interpolation techniques, specifically Franke-style interpolation, adapting hat-shaped basis functions to the AMR context to solve the "stitching" problem between adjacent cells of different resolutions [S1, unofficial]. It contrasts with earlier methods by Weber et al. and Moran and Ellsworth, which relied on subdividing coarse cells, and Leaf et al. and Ljung et al., which used truncated support functions [S1, unofficial][S2].

## Sources
[S1] CPU volume rendering of adaptive mesh refinement data, https://www.researchgate.net/publication/321321949_CPU_volume_rendering_of_adaptive_mesh_refinement_data, fetched 2026-10-26 (unofficial)
[S2] PDF Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
