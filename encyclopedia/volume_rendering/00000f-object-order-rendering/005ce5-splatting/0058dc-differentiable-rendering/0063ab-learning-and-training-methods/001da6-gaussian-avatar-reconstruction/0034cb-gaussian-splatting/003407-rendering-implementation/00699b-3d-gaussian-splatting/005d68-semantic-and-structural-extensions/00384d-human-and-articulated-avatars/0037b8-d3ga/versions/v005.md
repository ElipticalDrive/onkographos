# D3GA

Also called: D3GA

## Overview
D3GA is a method for creating animatable human avatars using drivable 3D Gaussians and tetrahedral cages for full-body modeling [S1]. It addresses the challenge of reconstructing dynamic humans from multi-view videos by providing promising geometry and appearance modeling [S2, S2, unofficial]. It sets itself apart from siblings like SplatArmor and HuGS by utilizing tetrahedral cages as the primary deformation structure for the drivable Gaussians [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Multi-view video data of a human subject; pose parameters defining the target animation.
*   **Outputs:** An animatable 3D Gaussian avatar that can be rendered from arbitrary viewpoints and poses.

**Data Structures:**
*   **3D Gaussians:** Drivable Gaussians initialized in a canonical space. Each Gaussian $i$ has attributes including position $\mathbf{p}_i$, rotation $\mathbf{q}_i$, scale $\mathbf{s}_i$, opacity $\alpha_i$, and color $\mathbf{c}_i$.
*   **Tetrahedral Cage:** A structural mesh composed of tetrahedra that encloses the human body. This cage defines the deformation field. The exact layout of the tetrahedral mesh (vertices, connectivity) is not specified in the sources [G: Standard tetrahedral mesh representation].

**Steps:**
1.  **Initialization:** Initialize 3D Gaussians in a canonical reference pose (e.g., T-pose) [G: Based on Shared principles].
2.  **Cage Construction:** Construct a tetrahedral cage around the canonical human geometry. This cage serves as the deformation driver [S1].
3.  **Deformation Mapping:** Map the canonical Gaussians to the target pose. The position of each Gaussian is deformed based on its location relative to the tetrahedral cage and the current pose parameters $\theta$. The deformation function $D(\cdot)$ maps a canonical Gaussian $i$ with position $\mathbf{p}_i^{canonical}$ to a target position $\mathbf{p}_i^{target}$:
    $$
    \mathbf{p}_i^{target} = D(\mathbf{p}_i^{canonical}, \theta)
    $$
    The rotation and scale of the Gaussian are also transformed by the deformation function to maintain physical consistency [G].
4.  **Rendering:** Render the deformed Gaussians using the standard differentiable rasterization pipeline. For a pixel $p$, the rendered color $\mathbf{C}(p)$ is computed by sorting overlapping deformed Gaussians $\mathcal{O}_p$ front-to-back and applying:
    $$
    \mathbf{C}(p) = \sum_{i \in \mathcal{O}_p} \mathbf{c}_i \cdot \alpha_i \cdot \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
    where $\mathbf{c}_i$ is the color, $\alpha_i$ is the opacity, and the product term is the transmittance [Shared principles].

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   Not given in the sources.

## Subtopics

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths and Limitations:** Provides promising geometry and appearance modeling for full-body human avatars [S1].
*   **When to use it:** When reconstructing dynamic humans from multi-view videos with a need for drivable 3D Gaussian representation [S1].
*   **When not:** Not given in the sources.

## Background
D3GA was introduced in the context of recent advances in 3D Gaussian splatting for digital human creation [S1]. It is cited as the first method to create animatable human avatars using drivable 3D Gaussians and tetrahedral cages [S1][S2, S2, unofficial]. It is referenced in the survey "Recent advances in 3D Gaussian splatting" [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
