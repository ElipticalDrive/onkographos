# Warp-based Dynamic Scene Representation

Also called: Deformation-based representations, Warp-based methods, Warping-based Dynamic Scene Representation

## Overview
Warp-based dynamic scene representations model temporal changes by learning a temporally global 3D canonical representation and a learnable, time-dependent deformation field that warps canonical space to match observed scenes at each timestep [S1]. Unlike [Space-time representations](6885-space-time-representations.md), which extend the query space to include time directly, warp-based methods explicitly separate geometry and appearance into a static canonical form and a dynamic transformation, ensuring long-term correspondence and smooth temporal interpolation [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D point $p$ in the deformed (observed) space, camera parameters $\mathcal{C}$, viewing direction $\mathbf{d}$, and time $t$.
*   **Output:** An implicit field value $\xi$ (e.g., density, color) at the queried location.

**Data Structures**
The method relies on two primary neural components:
1.  **Deformation Network ($T$):** A function mapping deformed points to canonical space. It is typically implemented as a Multi-Layer Perceptron (MLP) [S1]. Alternatively, it may be modeled explicitly using polynomial bases, Fourier transforms, or learned basis functions [S1].
2.  **Static Representation ($F$):** A function inferring the implicit field from canonical points. It takes the canonical point $p'$, direction $\mathbf{d}$, and camera parameters $\mathcal{C}$ as input [S1].

**Formulas**
The rendering function $f$ is defined as the composition of the deformation network and the static representation [S1]:

$$f = F \circ T$$

Where the deformation network $T$ maps a point $p$ and canonical camera configuration $\mathcal{C}_0$ to its canonical counterpart $p'$ [S1]:

$$T : (p, \mathcal{C}_0) \rightarrow p'$$

And the static representation $F$ infers the field from $p'$ [S1]:

$$F : p'[\mathbf{d}; \mathcal{C}] \rightarrow \xi$$

**Steps**
1.  **Learn Canonical Representation:** Train a static representation $F$ on a canonical space that represents the object or scene structure independent of time [S1].
2.  **Learn Deformation Field:** Train the deformation network $T$ to map observed 3D points $p$ from specific time steps to their corresponding locations in the canonical space [S1].
3.  **Warping Strategy:**
    *   **Backward Warp:** Map deformed states back to the canonical representation. This is the standard approach in Eq. 11 [S1].
    *   **Forward Warp:** Alternatively, learn a mapping from canonical space to dynamic time steps. This provides a smooth and continuous deformation mapping, mitigating discontinuities associated with backward warping [S1].
4.  **Rendering:** For a query at time $t$, use $T$ to warp the point to canonical space, then query $F$ to obtain the field value $\xi$ [S1].

**Pitfalls and Edge Cases**
*   **Discontinuity in Backward Warp:** When mapping 3D points back to canonical space across different time steps, the backward warp often results in discontinuous mappings because various object configurations can occupy the same spatial location [S1].
*   **Topology Changes:** Warp-based methods struggle with topology changes, limiting their applicability primarily to object-level modeling where the object maintains a consistent structure over time [S1].
*   **Implementation Note:** When using a backward warp, care must be taken to handle collisions or overlapping configurations in the deformed space that map to the same canonical coordinate [S1].

## Operating characteristics
*   **Strengths:** Ensures long-term correspondence between frames, allowing for consistent novel view synthesis and smooth interpolation in the time dimension [S1].
*   **Limitations:** Struggles with topology changes; not suitable for scenes with significant structural changes (e.g., breaking, merging) [S1].
*   **Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.

## Background
Warp-based methods were introduced as a class of dynamic scene representations that condition the representation with a time component, modeling geometry and appearance as a continuous function of both space and time [S1]. They differ from space-time representations by explicitly using a deformation network rather than extending the query space with time [S1].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
