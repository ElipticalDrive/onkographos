# Warp-based Dynamic Scene Representation

Also called: Deformation-based representations, Warp-based methods, Warping-based Dynamic Scene Representation

## Overview
Warp-based dynamic scene representations model temporal changes by learning a temporally global 3D canonical representation and a learnable, time-dependent deformation field that warps canonical space to match observed scenes at each timestep [S1]. Unlike [Space-time representations](6885-space-time-representations.md), which extend the query space to include time directly, warp-based methods explicitly separate geometry and appearance into a static canonical form and a dynamic transformation, ensuring long-term correspondence and smooth temporal interpolation [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D point $p$ in the deformed (observed) space, camera parameters $\mathcal{C}$, viewing direction $\mathbf{d}$, and time $t$ [S1].
*   **Output:** An implicit field value $\xi$ (e.g., density, color) at the queried location [S1].

**Data Structures**
The method relies on two primary neural components:
1.  **Deformation Network ($T$):** A function modeling the temporal deformation field, typically implemented as a Multi-Layer Perceptron (MLP) [S1]. Alternatively, it may be modeled explicitly using polynomial bases, Fourier transforms, or learned basis functions [S1].
2.  **Static Representation ($F$):** A function inferring the implicit field from canonical points, taking the canonical point $p'$, direction $\mathbf{d}$, and camera parameters $\mathcal{C}$ as input [S1].

**Formulas**
The rendering function $f$ is defined as the composition of the deformation network and the static representation [S1]:

$$f = F \circ T \text{ with } T : (p, \mathcal{C}_0) \rightarrow p' \text{ and } F : p'[\mathbf{d}; \mathcal{C}] \rightarrow \xi, \quad (11)$$

where the operator $\circ$ denotes function composition, $T$ maps point $p$ and canonical camera configuration $\mathcal{C}_0$ to its canonical counterpart $p'$, and $F$ infers the field from $p'$ [S1].

**Steps**
1.  **Learn Canonical Representation:** Train a static representation $F$ on a canonical space that represents the object or scene structure independent of time [S1].
2.  **Learn Deformation Field:** Train the deformation network $T$ to model the temporal mapping. This can be implemented as a backward warp representing a mapping of deformed states back to the canonical representation, or a forward warp providing a smooth and continuous deformation mapping from canonical space to dynamic time steps [S1].
3.  **Rendering:** For a query at time $t$, use $T$ to warp the input point $p$ to canonical space to obtain $p'$, then evaluate $F$ with $p'$, $\mathbf{d}$, and $\mathcal{C}$ to produce the output $\xi$ [S1].

**Pitfalls and Edge Cases**
*   **Discontinuity in Backward Warp:** When attempting to map 3D points back to the canonical space across different time steps, the backward warp often results in discontinuous mappings because various object configurations can occupy the same spatial location [S1].
*   **Topology Changes:** The methods struggle with topology changes, limiting their applicability primarily to object-level modeling where the object maintains a consistent structure over time [S1].

## Subtopics

### DeVRF
A method that designs a static-dynamic learning paradigm to simplify learning 4D voxel deformation fields using only a few dynamic sequences [S2]. It uses a 3D volume canonical space learned from multi-view static images [S2]. Model training is prone to overfitting when modeling dynamic non-rigid scenes with 4D deformation fields on discrete voxels due to network parameter constraints [S2].

## Operating characteristics
*   **Strengths:** Ensures long-term correspondence between frames, allowing for consistent novel view synthesis and smooth interpolation in the time dimension [S1].
*   **Limitations:** Struggles with topology changes; not suitable for scenes with significant structural changes (e.g., breaking, merging) [S1].
*   **Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.

## Background
Warp-based methods were introduced as a class of dynamic scene representations that condition the representation with a time component, modeling geometry and appearance as a continuous function of both space and time [S1]. They differ from space-time representations by explicitly using a deformation network rather than extending the query space with time [S1].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
