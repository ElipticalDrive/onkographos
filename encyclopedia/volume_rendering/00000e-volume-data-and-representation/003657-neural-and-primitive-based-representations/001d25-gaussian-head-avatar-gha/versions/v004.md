# Gaussian Head Avatar (GHA)
## Overview
Gaussian Head Avatar (GHA) is a neural primitive-based representation for dynamic head avatars that predicts 3D Gaussian positions based on facial expression and head pose, alongside a color MLP [S1][S2, unofficial]. Unlike Gaussian Avatars, which optimizes Gaussians constrained to a FLAME mesh, GHA uses a feedforward architecture with two MLPs and a deformation module to generate the Gaussian field [S1][S2, unofficial].
## How it works
**Inputs:**
- Facial expression latent code $\mathbf{e}$ [G].
- Head pose latent code $\mathbf{p}$ [G].
- 3D spatial coordinate $\mathbf{x}$ [G].
- View direction $\mathbf{d}$ [G].

**Outputs:**
- 3D Gaussian positions [S1][S2, unofficial].
- Color and opacity parameters [G].

**Data Structures:**
- Two Multi-Layer Perceptrons (MLPs) and a deformation module [S1][S2, unofficial].
- A super-resolution network [S3, unofficial].

**Steps:**
1. Encode the input expression $\mathbf{e}$ and pose $\mathbf{p}$ into a latent representation [G].
2. Pass $(\mathbf{e}, \mathbf{p}, \mathbf{x})$ through the deformation module and position MLP to predict the 3D Gaussian positions [S1][S2, unofficial].
3. Pass $(\mathbf{e}, \mathbf{p}, \mathbf{x}, \mathbf{d})$ through the color MLP to predict color and opacity [G].
4. Apply a super-resolution network to achieve high-fidelity head avatar learning [S3, unofficial].
5. Render the predicted Gaussians using standard 3D Gaussian Splatting rasterization [G].

**Parameters:**
- Not given in the sources.

**Edge cases and pitfalls:**
- Not given in the sources.

## Subtopics
## Operating characteristics
**Time and memory complexity:**
- Memory: Constant with respect to spatial resolution, determined by the MLP size [G].
- Time: Not given in the sources.

**Measured performance:**
- Not given in the sources.

**Strengths and limitations:**
- Strengths: Not given in the sources.
- Limitations: Relies on the accuracy of the expression/pose encoding [G].

**When to use it:**
- Use for dynamic head avatars requiring real-time rendering of novel expressions [G].
- Do not use if high-fidelity static scene reconstruction is the primary goal without dynamic variation [G].

## Background
GHA was introduced as an alternative to Gaussian Avatars, which optimizes Gaussians constrained to a FLAME mesh [S1][S2, unofficial]. GHA leverages MLPs to predict Gaussian positions based on expression and pose, alongside a color MLP [S1][S2, unofficial]. It is part of the broader family of Gaussian Avatar methods, including Gaussian Avatar (SMPL-based) and 3DGS-Avatar [S1][S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
[S3] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
