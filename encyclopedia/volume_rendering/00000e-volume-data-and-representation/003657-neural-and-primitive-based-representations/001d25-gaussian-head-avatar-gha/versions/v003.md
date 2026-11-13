# Gaussian Head Avatar (GHA)
## Overview
Gaussian Head Avatar (GHA) is a neural primitive-based representation for dynamic head avatars that predicts 3D Gaussian parameters directly from facial expression and head pose latent codes [S1][S2, unofficial]. Unlike methods that optimize static Gaussians per scene or attach primitives to mesh triangles, GHA uses a feedforward network to generate a continuous field of Gaussians, enabling real-time rendering of novel expressions and poses without per-scene optimization [S1][S2, unofficial].
## How it works
**Inputs:**
- Facial expression latent code $\mathbf{e}$ (typically from a parametric model like FLAME).
- Head pose latent code $\mathbf{p}$ (rotation and translation).
- 3D spatial coordinate $\mathbf{x}$ for evaluation.

**Outputs:**
- Gaussian parameters: position $\mathbf{p}_g$, covariance $\mathbf{\Sigma}$, opacity $\alpha$, and spherical harmonic coefficients for color $\mathbf{c}$.

**Data Structures:**
- Two Multi-Layer Perceptrons (MLPs):
  1. **Deformation/Position MLP:** Maps $(\mathbf{e}, \mathbf{p}, \mathbf{x})$ to 3D Gaussian position and covariance.
  2. **Color MLP:** Maps $(\mathbf{e}, \mathbf{p}, \mathbf{x}, \mathbf{d})$ to opacity and color, where $\mathbf{d}$ is the view direction.
- Note: The sources specify "two MLPs and a deformation module" [S1][S2, unofficial]. The deformation module is integrated into the position prediction pipeline.

**Steps:**
1. Encode the input expression $\mathbf{e}$ and pose $\mathbf{p}$ into a latent representation.
2. For a query point $\mathbf{x}$, pass $(\mathbf{e}, \mathbf{p}, \mathbf{x})$ through the deformation/position MLP to predict the Gaussian mean $\mathbf{\mu}$ and covariance $\mathbf{\Sigma}$ [S1][S2, unofficial].
3. Pass $(\mathbf{e}, \mathbf{p}, \mathbf{x}, \mathbf{d})$ through the color MLP to predict opacity $\alpha$ and color $\mathbf{c}$ [S1][S2, unofficial].
4. Render the predicted Gaussians using standard 3D Gaussian Splatting rasterization [G].

**Parameters:**
- Not given in the sources.

**Edge cases and pitfalls:**
- Not given in the sources.

## Subtopics
## Operating characteristics
**Time and memory complexity:**
- Memory: Constant with respect to spatial resolution, determined by the MLP size [G].
- Time: Real-time rendering is enabled by the feedforward nature, avoiding per-scene optimization [S1][S2, unofficial].

**Measured performance:**
- Not given in the sources.

**Strengths and limitations:**
- Strengths: Generalizes to novel expressions and poses; no per-scene optimization required [S1][S2, unofficial].
- Limitations: Relies on the accuracy of the expression/pose encoding [G].

**When to use it:**
- Use for dynamic head avatars requiring real-time rendering of novel expressions.
- Do not use if high-fidelity static scene reconstruction is the primary goal without dynamic variation.

## Background
GHA was introduced as an alternative to Gaussian Avatars, which optimizes Gaussians constrained to a FLAME mesh [S1][S2, unofficial]. GHA leverages MLPs to predict Gaussian positions based on expression and pose, alongside a color MLP [S1][S2, unofficial]. It is part of the broader family of Gaussian Avatar methods, including Gaussian Avatar (SMPL-based) and 3DGS-Avatar [S1][S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
