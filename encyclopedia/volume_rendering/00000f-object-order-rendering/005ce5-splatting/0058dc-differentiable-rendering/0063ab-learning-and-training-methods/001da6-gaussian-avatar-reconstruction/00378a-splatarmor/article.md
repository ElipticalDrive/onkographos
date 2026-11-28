# SplatArmor
## Overview
SplatArmor reconstructs animatable human avatars using drivable 3D Gaussians [S1][S2, unofficial]. It sets itself apart by leveraging two different MLPs to predict large motions built upon the SMPL and canonical space, and by allowing pose-dependent effects through the use of SE(3) fields [S1][S2, unofficial].
## How it works
**Inputs and Outputs:**
*   **Inputs:** Not given in the sources.
*   **Outputs:** Not given in the sources.
**Data Structures:**
*   **Gaussian Attributes:** Position, covariance, opacity, and spherical harmonics coefficients for color [G].
*   **SMPL Model:** Parametric human body model providing a canonical mesh and skinning weights [G].
*   **MLPs:** Two separate neural networks used for motion prediction [S1][S2, unofficial].
*   **SE(3) Fields:** A field representation encoding pose-dependent transformations [S1][S2, unofficial].
**Method Steps:**
1.  **Large Motion Prediction:** Leverage two different MLPs to predict large motions built upon the SMPL and canonical space [S1][S2, unofficial].
2.  **Pose-Dependent Effects:** Allow pose-dependent effects through the use of SE(3) fields [S1][S2, unofficial].
3.  **Deformation & Rendering:** Apply the predicted motions and SE(3) transformations to the Gaussians, then render using differentiable 3D Gaussian Splatting [G].
**Formulas:** Not given in the sources.
**Parameters:** Not given in the sources.
**Edge Cases and Pitfalls:** Not given in the sources.
## Operating characteristics
*   **Performance:** Not given in the sources.
*   **Strengths:** Enables more detailed geometry and appearance modeling [S1][S2, unofficial].
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.
## Background
SplatArmor is discussed in recent surveys on 3D Gaussian Splatting advances [S1][S2, unofficial]. It is cited as a method that leverages two different MLPs and SE(3) fields to capture more dynamic details and pose-dependent effects [S1][S2, unofficial].
## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
