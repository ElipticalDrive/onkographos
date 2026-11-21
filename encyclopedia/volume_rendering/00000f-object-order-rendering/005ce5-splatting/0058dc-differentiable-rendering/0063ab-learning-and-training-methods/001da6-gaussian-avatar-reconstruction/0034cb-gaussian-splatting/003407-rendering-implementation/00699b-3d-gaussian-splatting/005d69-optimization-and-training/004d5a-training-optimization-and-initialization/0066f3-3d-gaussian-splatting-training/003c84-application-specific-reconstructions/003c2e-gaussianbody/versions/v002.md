# GaussianBody


## Overview
GaussianBody is a method for dynamic digital human modeling from monocular video that leverages physically-based priors to regularize 3D Gaussians in canonical space [S1][S2, unofficial]. It solves the problem of visual artifacts in dynamic cloth reconstruction caused by the ambiguity of monocular input [S2, unofficial]. It sets itself apart from standard 3DGS by enforcing geometric consistency in the canonical domain rather than relying solely on photometric loss in the image domain [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Monocular video sequence of a human subject; a parametric body model (e.g., SMPL) providing canonical space geometry [S1][S2, unofficial].
- Output: A set of 3D Gaussians with attributes (position, covariance, opacity, spherical harmonics) that can be rendered into novel views and poses [S1][S2, unofficial].

**Data Structures:**
- Gaussians are stored in canonical space. Each Gaussian $i$ has attributes: mean $\mu_i$, covariance $\Sigma_i$, opacity $\alpha_i$, and color coefficients $c_i$ [S1][S2, unofficial].
- The parametric model provides a mesh surface $M$ in canonical space [S2, unofficial].

**Steps:**
1. **Initialization:** Initialize Gaussians based on the parametric body model in canonical space [S1][S2, unofficial].
2. **Deformation Prediction:** Predict the deformation from canonical space to the observed pose space using a deformation network or field [S1][S2, unofficial].
3. **Regularization:** Apply physically-based priors to regularize the Gaussians in canonical space. This involves constraining the Gaussians to adhere to the geometry priors of the parametric model to ensure pose-dependent cloth deformation [S1][S2, unofficial].
4. **Rendering:** Transform Gaussians from canonical to view space based on the predicted deformation and camera parameters. Render the image using standard 3DGS rasterization [S1][S2, unofficial].
5. **Optimization:** Minimize a combined loss function:
   $$ \mathcal{L} = \mathcal{L}_{photo} + \mathcal{L}_{phys} $$
   where $\mathcal{L}_{photo}$ is the photometric reconstruction loss and $\mathcal{L}_{phys}$ is the physical prior loss enforcing geometric consistency in canonical space [S1][S2, unofficial].

**Parameters:**
- Weight of physical prior loss $\lambda_{phys}$: Not given in the sources.
- Learning rate: Not given in the sources.

**Edge Cases and Pitfalls:**
- Monocular ambiguity can lead to artifacts in dynamic cloth if physical priors are not strong enough [S2, unofficial].
- Precision issues in canonical space deformation prediction may cause misalignment [G].

## Subtopics

### HAHA
HAHA attaches Gaussians to the surface of a mesh and combines rendered results from a textured human body mesh and Gaussians to reduce the number of Gaussians required [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Avoids artifacts in dynamic cloth from monocular video; generalizes to novel poses effectively [S1][S2, unofficial].
- **Limitations:** Relies on the accuracy of the parametric body model and physically-based priors [S1][S2, unofficial].
- **When to use:** Dynamic digital human modeling from monocular video where cloth artifacts are a concern [S1][S2, unofficial].
- **When not to use:** When high-fidelity static geometry is sufficient or when multi-view data is available [G].

## Background
GaussianBody was introduced in [214] (cited as [S1][S2, unofficial]) for dynamic digital human modeling based on monocular video. It builds on 3DGS by incorporating physically-based priors to regularize Gaussians in canonical space [S1][S2, unofficial]. It is part of the family of application-specific reconstructions that modify the 3DGS optimization loop with domain-specific constraints [Parent].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
