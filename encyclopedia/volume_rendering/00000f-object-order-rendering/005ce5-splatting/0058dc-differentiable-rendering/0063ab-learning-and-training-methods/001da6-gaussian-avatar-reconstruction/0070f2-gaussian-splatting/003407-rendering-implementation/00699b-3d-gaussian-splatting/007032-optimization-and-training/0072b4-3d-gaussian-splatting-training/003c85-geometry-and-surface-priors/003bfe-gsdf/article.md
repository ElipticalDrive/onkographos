# GSDF
Also called: Gaussian SDF, Gaussian Signed Distance Function

## Overview
Builds on [Geometry and Surface Priors](9270-geometry-and-surface-priors.md). GSDF is a training optimization technique that aligns the discrete 3D Gaussian representation with a continuous Signed Distance Function (SDF) prior to enhance geometry reconstruction quality [S1]. It sets itself apart from sibling methods like [NeuSG](9172-neusg.md) by explicitly transferring surface properties from a neural SDF to the Gaussians [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D Gaussian parameters (centers, covariances, opacities, colors), a neural SDF field, and input images with camera poses [G].
*   **Outputs:** Optimized 3D Gaussians suitable for rendering and mesh extraction [G].

**Data Structures:**
*   **Neural SDF:** A continuous field $S(\mathbf{x})$ mapping 3D coordinates to signed distance values [G].
*   **Gaussian Attributes:** Standard 3DGS attributes including position, covariance, opacity, and spherical harmonics for color [G].

**Method Steps:**
1.  **Render:** Perform standard differentiable volume rendering to generate predicted images from the current Gaussian set [G].
2.  **Compute Photometric Loss:** Calculate $\mathcal{L}_{rgb}$ between rendered and input images [G].
3.  **Evaluate SDF:** Query the neural SDF at each Gaussian center $\mathbf{c}_i$ to obtain $S(\mathbf{c}_i)$ [G].
4.  **Compute Geometric Loss:** Calculate $\mathcal{L}_{geo}$ to encourage Gaussian centers to lie on the zero-level set of the SDF [S1].
5.  **Compute Normal Loss:** Calculate $\mathcal{L}_{norm}$ to align Gaussian orientations with the surface normals derived from the SDF gradient [S1].
6.  **Optimize:** Update Gaussian parameters and SDF weights by minimizing the total loss [S1].

**Formulas:**
The total loss function combines photometric and geometric terms:
$$ \mathcal{L}_{total} = \mathcal{L}_{rgb} + \lambda_{geo} \mathcal{L}_{geo} + \lambda_{norm} \mathcal{L}_{norm} $$
where $\mathcal{L}_{geo}$ enforces positional consistency by minimizing the absolute SDF value at Gaussian centers, and $\mathcal{L}_{norm}$ enforces normal consistency by aligning Gaussian orientations with the SDF gradient [S1]. The weights $\lambda_{geo}$ and $\lambda_{norm}$ balance photometric fidelity against geometric accuracy [G].

**Parameters:**
*   $\lambda_{geo}, \lambda_{norm}$: Hyperparameters controlling the weight of geometric and normal consistency losses [G]. Typical values are not specified in the sources.

**Edge Cases:**
*   Not given in the sources.

## Operating characteristics
*   **Complexity:** Adds computational overhead proportional to the number of Gaussians for querying the neural SDF during each training iteration [G].
*   **Strengths:** Recovers more detailed mesh structures compared to standard 3DGS [S2, unofficial]. Enhances overall geometry reconstruction quality [S1].
*   **Limitations:** Requires a reliable and accurately trained neural SDF prior [G].
*   **When to use:** When high-quality surface reconstruction and mesh extraction are required from 3DGS [S1][S2, unofficial].

## Background
GSDF was introduced to leverage neural signed distance functions (SDFs) for surface reconstruction, inspired by similar approaches used in NeRF-based methods [S2, unofficial]. It is closely related to methods like 3DGSR that also encourage consistency between SDF and 3DGS representations [S1]. The approach was later extended in works like 2DGS, which replaced 3D Gaussians with 2D Gaussians to improve rendering accuracy [S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
