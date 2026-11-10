# Warp-based Dynamic Scene Representation

Also called: Deformation-based representations, Warp-based methods, Warping-based Dynamic Scene Representation

## Overview
Warp-based dynamic scene representations model temporal changes by learning a temporally global 3D canonical representation and a learnable, time-dependent deformation field to warp canonical space to match observed scenes at each timestep [S1]. This approach ensures long-term correspondence, allowing for consistent novel view synthesis and smooth interpolation in the time dimension [S1]. Unlike [Space-time representations](6885-space-time-representations.md), which extend the query space to include time directly, warp-based methods explicitly separate geometry and appearance into a static component and a deformation component [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D point $p$ in the deformed (observed) space, camera parameters $\mathcal{C}$, viewing direction $\mathbf{d}$, and time $t$.
*   **Output:** An implicit field value $\xi$ (e.g., density, color) at the queried location.

**Data Structures:**
*   **Deformation Network $T$:** A function mapping points from the deformed space to the canonical space (backward warp) or vice versa (forward warp). It is typically implemented as a Multi-Layer Perceptron (MLP) [S1]. Alternatively, it can be modeled explicitly using polynomial bases, Fourier transforms, or learned basis functions [S1].
*   **Static Representation $F$:** A function that infers the implicit field from the canonical point $p'$. This is often a static NeRF-like representation or a voxel grid [S1].

**Formulas:**
The rendering function $f$ is defined as the composition of the deformation network $T$ and the static representation $F$ [S1]:

$$f = F \circ T$$

Specifically, the mapping is formalized as [S1]:

$$T : (p, \mathcal{C}_0) \rightarrow p'$$
$$F : p'[\mathbf{d}; \mathcal{C}] \rightarrow \xi$$

Where:
*   $T$ is the deformation network mapping point $p$ to its canonical counterpart $p'$.
*   $F$ is the static representation inferring the implicit field from $p'$.
*   $\mathcal{C}_0$ is the canonical camera configuration.
*   $\mathbf{d}$ is the viewing direction.
*   $\mathcal{C}$ is the camera parameters.
*   $\xi$ is the output field value.

**Steps:**
1.  **Learn Canonical Representation:** Learn a temporally global 3D representation (canonical space) that captures the static geometry and appearance of the scene [S1].
2.  **Learn Deformation Field:** Learn a deformation field $T$ that maps points between the canonical space and the deformed space at specific time steps [S1].
    *   **Backward Warp:** Map deformed states back to the canonical representation ($p \rightarrow p'$) [S1].
    *   **Forward Warp:** Alternatively, map from canonical space to dynamic time steps ($p' \rightarrow p$) to ensure smooth and continuous deformation [S1].
3.  **Render:** Query the static representation $F$ at the warped canonical point $p'$ to obtain the field value $\xi$ [S1].

**Pitfalls and Edge Cases:**
*   **Discontinuous Mapping:** Backward warps often result in discontinuous mappings because various object configurations can occupy the same spatial location at different time steps [S1]. Forward warps are preferred to mitigate this [S1].
*   **Topology Changes:** Warp-based methods struggle with topology changes, limiting their applicability primarily to object-level modeling where the object maintains a consistent structure over time [S1].

## Subtopics

### [DeVRF](8613-devrf.md)
DeVRF adapts NeRF-like volume rendering frameworks with explicit 3D representations to dynamic scenes by combining an explicit voxel-based canonical 3D volumetric grid with a 4D deformation field. It solves the problem of efficiently learning non-rigid motion in dynamic scenes by employing a static-to-dynamic learning paradigm: first learning the static scene representation from multi-view stills, then learning the deformation from a few-view dynamic sequence. This approach allows for much faster convergence than fully implicit networks and simplifies the learning of 4D voxel deformation fields using limited dynamic data.

## Operating characteristics
*   **Strengths:** Ensures long-term correspondence, allowing consistent novel view synthesis and smooth interpolation in the time dimension [S1].
*   **Limitations:** Struggles with topology changes, limiting applicability to object-level modeling with consistent structure [S1]. Backward warps can cause discontinuities [S1].
*   **Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.

## Background
Warp-based methods were introduced as a class of space-time representations that condition the representation with a time component [S1]. They model geometry and appearance as a continuous function of both space and time [S1]. Key references include [PSB* 21, PSH* 21, TTG* 21, LDZY24, WYF* 24, YGZ* 24] for warping-based methods and [GSD* 23] for forward warp approaches [S1].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
