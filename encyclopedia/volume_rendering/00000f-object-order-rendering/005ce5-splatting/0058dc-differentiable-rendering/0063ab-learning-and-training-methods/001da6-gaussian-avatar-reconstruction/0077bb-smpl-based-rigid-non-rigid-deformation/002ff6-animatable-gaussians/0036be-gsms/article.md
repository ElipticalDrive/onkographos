# GSMs

Also called: Gaussian Shell Maps (GSMs), Gaussian Splatting for Humans

## Overview
Gaussian Shell Maps (GSMs) is a method for generating animatable 3D humans by combining CNN-based generators with 3D Gaussian Splatting (3DGS) to recreate virtual humans with sophisticated details such as clothing and accessories [S2, unofficial]. It builds multi-layer shells from the SMPL template and binds 3D Gaussians to these shells [S1][S2, unofficial]. By utilizing the differentiable rendering of 3DGS and the generative adversarial network of StyleGAN2, animatable 3D humans are efficiently generated [S1][S2, unofficial]. It is distinguished from sibling methods like GAvatar, which attaches Gaussians to SMPL-X primitives, and HumanGaussian, which initializes Gaussians via random sampling [S1][S2, unofficial].

## How it works
**Inputs:** Multi-view images of a human subject; SMPL template mesh [S1][S2, unofficial].
**Outputs:** A set of 3D Gaussians bound to multi-layer shells, parameterized for animation [S1][S2, unofficial].

**Data Structures:**
1.  **SMPL Template:** A parametric human body model used as a geometric prior [S1][S2, unofficial].
2.  **Multi-layer Shells:** A set of surfaces constructed from the SMPL template [S1][S2, unofficial].
3.  **3D Gaussians:** Bound to the shells [S1][S2, unofficial].
4.  **StyleGAN2 Generator:** A CNN-based generator used to predict Gaussian attributes [S1][S2, unofficial].

**Steps:**
1.  **Shell Construction:** Build multi-layer shells from the SMPL template [S1][S2, unofficial].
2.  **Gaussian Binding:** Bind 3D Gaussians to the shells [S1][S2, unofficial].
3.  **Attribute Prediction:** Use a CNN-based generator (StyleGAN2) to predict the attributes of the 3D Gaussians [S2, unofficial].
4.  **Rendering:** Utilize the differentiable rendering of 3DGS to render the scene [S1][S2, unofficial].
5.  **Training:** Train the system using the generative adversarial network of StyleGAN2 [S1][S2, unofficial].

**Parameters:**
*   Number of shells: Not given in the sources.
*   StyleGAN2 configuration: Not given in the sources.

**Edge Cases and Pitfalls:**
*   Precision: Not given in the sources.
*   Boundaries: Not given in the sources.
*   Degenerate input: Not given in the sources.

## Subtopics

### SMPL-based rigid/non-rigid deformation
*   **What it is:** A deformation mechanism using the SMPL model to drive the avatar.
*   **What sets it apart:** Uses SMPL as a model prior to assist in generation.
*   **When to choose:** When a parametric human body model is required for animation.
*   **Link:** [SMPL-based rigid/non-rigid deformation](6158-smpl-based-rigid-non-rigid-deformation.md)

### Animatable Gaussians
*   **What it is:** An avatar method using SMPL-X rig with explicit pose-conditioned offsets.
*   **What sets it apart:** Incorporates 3DGS and 2D CNNs for more accurate human appearance and realistic garment dynamics using a template-guided parameterization and pose projection mechanism.
*   **When to choose:** When realistic garment dynamics and accurate appearance are prioritized.
*   **Link:** [Animatable Gaussians](6157-animatable-gaussians.md)

## Operating characteristics
*   **Time complexity:** Not given in the sources.
*   **Memory complexity:** Not given in the sources.
*   **Measured performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Efficient generation of animatable 3D humans; recreation of sophisticated details such as clothing and accessories [S1][S2, unofficial].
*   **Limitations:** Not given in the sources.
*   **When to use:** When generating virtual humans with detailed clothing and accessories is required [S2, unofficial].
*   **When not:** Not given in the sources.

## Background
GSMs were introduced as a special category of human body modeling that introduces a model prior, such as SMPL, to assist in generation [S1][S2, unofficial]. The method builds multi-layer shells from the SMPL template and binds 3D Gaussians to the shells [S1][S2, unofficial]. By utilizing the differentiable rendering of 3DGS and the generative adversarial network of StyleGAN2, animatable 3D humans are efficiently generated [S1][S2, unofficial]. It is cited as reference [211] in the source literature [S1][S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
