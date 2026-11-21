# Forward Flow for Novel View Synthesis of Dynamic Scenes

Also called: ForwardFlowDNeRF

## Overview
Forward Flow is a dynamic NeRF method that replaces backward flow deformation with forward flow to warp a canonical radiance field to target time steps, ensuring smooth and continuous motion modeling [S1]. Unlike backward flow methods that map points from current time to canonical space (causing discontinuity for fixed 3D positions), forward flow maps canonical space to current time, yielding smooth flow norms for the same 3D position along the timeline [S1].

## How it works
**Inputs:** A set of multi-view RGB images of a dynamic scene at various time steps $t$.
**Outputs:** A novel view synthesis at any arbitrary time $t$ and camera pose.

**Data Structures:**
1.  **Canonical Radiance Field ($V_{grid}^{Can}$):** Represented as a voxel grid (finite and discrete) rather than a continuous MLP. This enables explicit spatial warping [S1].
2.  **Deformation Feature Grid ($V_{grid}^{Df}$):** A grid storing deformation features for the canonical space [S1].
3.  **Lightweight MLP ($F_\theta$):** Used to model the canonical trajectory field [S1].

**Steps:**
1.  **Forward Flow Estimation:** For a query 3D point $\mathbf{p}$ in canonical space and a target time $t$, the deformation flow $\mathbf{V}_{flow}$ from canonical to time $t$ is estimated. This is done by querying the deformation feature grid $V_{grid}^{Df}$ at $\mathbf{p}$ and passing it through the lightweight MLP $F_\theta$ [S1].
    $$ \mathbf{V}_{flow} = F_\theta(V_{grid}^{Df}(\mathbf{p})) $$
2.  **Differential Forward Warping:** The canonical radiance field $V_{grid}^{Can}$ is warped to time $t$ to create an intermediate radiance field $V_{grid}^t$. This step addresses two mapping issues inherent in forward warping:
    *   **Many-to-One Mapping:** Multiple positions in the source (canonical) correspond to the same position in the target (time $t$). This is resolved using an **average splatting operation** [S1].
    *   **One-to-Many Mapping:** One position in the source corresponds to multiple positions in the target, causing holes/gaps. This is resolved using an **inpaint network** [S1].
    The warped field is denoted as $V_{grid}^{t, Inp}$ after inpainting [S1].
3.  **Volume Rendering:** Novel views at time $t$ are rendered via standard volume rendering on the inpainted radiance field $V_{grid}^{t, Inp}$ [S1].

**Pitfalls:**
*   **Many-to-One Mapping:** Causes occlusion/overlap artifacts if not handled by average splatting [S1].
*   **One-to-Many Mapping:** Causes holes/gaps in the warped field if not handled by inpainting [S1].
*   **Backward Flow Discontinuity:** Backward flow fields are non-smooth and discontinuous for fixed 3D positions along the timeline, making them difficult to fit with smooth motion models (like MLPs) and introducing distortions [S1]. Forward flow avoids this by mapping canonical space to current time, which is smooth and continuous [S1].

## Operating characteristics
Not given in the sources.

## Background
Forward Flow was proposed to address the limitations of backward flow fields in dynamic NeRFs. Backward flow fields are non-smooth and discontinuous, leading to distortions in the canonical radiance field due to the limited capacity of motion models to fit such fields [S1]. The method was introduced in 2023 [S1].

## Sources
[S1] Forward Flow for Novel View Synthesis of Dynamic Scenes, https://doi.org/10.1109/iccv51070.2023.01468, fetched 2026-10-26
