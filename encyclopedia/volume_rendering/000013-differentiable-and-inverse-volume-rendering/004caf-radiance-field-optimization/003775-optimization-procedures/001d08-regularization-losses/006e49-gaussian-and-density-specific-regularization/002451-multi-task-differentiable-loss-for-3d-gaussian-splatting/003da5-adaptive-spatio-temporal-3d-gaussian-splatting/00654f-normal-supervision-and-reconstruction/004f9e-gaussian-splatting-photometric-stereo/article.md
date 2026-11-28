# Gaussian Splatting Photometric Stereo

Also called: GS-PS

## Overview
Gaussian Splatting Photometric Stereo (GS-PS) is a calibrated photometric stereo method that represents the scene exclusively using 3D Gaussian primitives to estimate surface normals and albedo [S1][S2, unofficial]. It solves the inverse problem of recovering geometry from multi-view images under varying illumination by optimizing Gaussian parameters (position, covariance, color, opacity) to minimize a combined photometric and normal alignment loss [S2, unofficial]. Unlike deep learning-based approaches that map images directly to normals, GS-PS optimizes an explicit physical image formation model, offering greater explainability and geometric interpretability of the optimized parameters [S1][S2, unofficial]. It builds on [Normal Supervision and Reconstruction](19339-normal-supervision-and-reconstruction.md) by explicitly integrating normal supervision into the Gaussian Splatting optimization loop.

## How it works
**Inputs:** A set of $N$ calibrated images of a scene taken under different known light directions. Each image $I_k$ corresponds to a specific light source direction $\mathbf{l}_k$.
**Outputs:** A set of 3D Gaussians representing the scene, where each Gaussian $i$ is defined by position $\mathbf{p}_i$, covariance matrix $\mathbf{\Sigma}_i$ (or rotation $\mathbf{R}_i$ and scale $\mathbf{s}_i$), opacity $\alpha_i$, spherical harmonic coefficients for view-dependent color, and a surface normal $\mathbf{n}_i$.

**Data Structures:**
Each Gaussian $i$ stores:
- $\mathbf{p}_i \in \mathbb{R}^3$: Center position.
- $\mathbf{R}_i \in \mathbb{R}^{3 \times 3}$: Rotation matrix (derived from quaternion).
- $\mathbf{s}_i \in \mathbb{R}^3$: Scale factors.
- $\alpha_i \in [0, 1]$: Opacity.
- $\mathbf{c}_i$: Color/albedo parameters (spherical harmonics).
- $\mathbf{n}_i \in \mathbb{R}^3$: Surface normal vector (unit length). [G: In standard 3DGS, normals are not explicit; here they are either optimized directly or derived from the covariance structure, but GS-PS treats them as explicit targets for supervision].

**Rendering Pipeline:**
1. **Normal Map Rendering:** For a given camera view, Gaussians are sorted by depth. The rendered normal map $\hat{\mathbf{n}}_{uv}$ at pixel $(u,v)$ is computed via alpha-blending:
   $$ \hat{\mathbf{n}}_{uv} = \sum_{i \in \text{sorted}} \mathbf{n}_i \alpha_i \prod_{j < i} (1 - \alpha_j) $$
   where $\mathbf{n}_i$ is the normal of the $i$-th Gaussian, $\alpha_i$ is its opacity, and the product term represents the transparency of preceding Gaussians [S2, unofficial].

2. **Photometric Rendering:** The rendered color $C_{uv}$ at pixel $(u,v)$ is computed using the standard 3DGS alpha-blending of colors, modulated by a lighting model. Since standard 3DGS does not account for lighting, GS-PS uses a Lambertian model to calculate self-occlusion shadows based on the alignment between the surface normal $\mathbf{n}_i$ and the light direction $\mathbf{l}_k$ [S2, unofficial]. The composite image $c(x, y)$ from multiple input images is often derived as the maximum intensity:
   $$ c(x, y) = \max\{I_1 (x, y), I_2 (x, y), \dots , I_n (x, y)\} $$
   [S2, unofficial].

**Optimization Steps:**
1. **Initialization:** Initialize Gaussians from depth estimates or sparse points. [G: Standard 3DGS initialization from SfM points is assumed].
2. **Rendering:** Render RGB images and normal maps for all input views under their respective light directions.
3. **Loss Calculation:**
   - **Photometric Loss ($L_c$):** Compute the L1 difference between the rendered image and the real input image [S1][S2, unofficial].
   - **Normal Regularization Loss ($L_n$):** Compute a target normal map $\mathbf{n}_{uv}^{target}$ from the gradient of an estimated depth map [S1][S2, unofficial]. Compare this with the rendered normal map $\hat{\mathbf{n}}_{uv}$ using a cosine distance or similarity loss [S1][S2, unofficial].
   - **Total Loss:**
     $$ L = L_c + \lambda L_n $$
     where $\lambda$ is the weighting factor for the normal regularization term [S1][S2, unofficial].
4. **Gradient Descent:** Optimize Gaussian parameters (position, covariance, color, opacity, and normals) using gradient descent [S1][S2, unofficial].
5. **Adaptive Density Control:**
   - **Pruning:** Remove Gaussians that are too large or have opacity below a threshold [S1][S2, unofficial].
   - **Cloning/Splitting:** Clone or split Gaussians in regions requiring more detail [S1][S2, unofficial].
6. **Iteration:** Repeat steps 2-5 until convergence.

**Parameters:**
- $\lambda$: Weighting factor for the normal regularization term. Typical value not given in sources. [S1][S2, unofficial].

**Edge Cases and Pitfalls:**
- **Cast Shadows:** The current model does not account for cast shadows (ombres portées). Errors are concentrated in regions where cast shadows are present on many input images, leading to high angular estimation errors [S1][S2, unofficial].
- **Lambertian Assumption:** The method assumes Lambertian materials. Performance degrades on specular surfaces unless extended [S1].

## Subtopics

### Optimization with Stochastic Solids
A variant that optimizes scenes using photometric loss, normal consistency loss, and multi-view regularization [S3, unofficial]. It retains the standard Gaussian Splatting approximation for RGB and normals but computes depth using a stochastic solid method (Eq. 10 in [S3, unofficial]) to provide denser supervision for optimization [S3, unofficial]. Experiments are conducted at half-resolution images [S3, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Rendering is efficient due to the explicit Gaussian representation, allowing fast gradient calculation [S1][S2, unofficial].
- Full volumetric rendering for all modalities is computationally expensive, so standard approximations are used for RGB and normals [S3, unofficial].

**Measured Performance:**
- On the DiliGenT dataset (Bear, Cat, Pot1 objects with Lambertian materials), GS-PS achieved mean angular errors of:
  - Cat: 8.13° [S1][S2, unofficial]
  - Bear: 10.11° [S2, unofficial] (Note: [S1] reports 8.89° for Bear, indicating a discrepancy or different run) [S1][S2, unofficial].
  - Pot1: 4.12° [S1][S2, unofficial]
- Compared to Baseline [12] (Woodham), DANINet [7], and Uni MS-PS [2], GS-PS shows competitive or comparable results, particularly on Pot1 [S1][S2, unofficial].

**Strengths and Limitations:**
- **Strengths:** More explainable than deep neural network methods because it optimizes an explicit physical image formation model [S1][S2, unofficial]. More explainable than inverse rendering methods using small networks because optimized parameters have a geometric interpretation [S1][S2, unofficial]. Efficient gradient calculation [S1][S2, unofficial].
- **Limitations:** Does not account for cast shadows, leading to high errors in shadowed regions [S1][S2, unofficial]. Assumes Lambertian materials [S1].

**When to Use:**
- Use for calibrated photometric stereo on Lambertian scenes where explainability and explicit geometry are desired.
- Avoid when cast shadows are prominent or materials are highly specular.

## Background
The method is based on the Gaussian Splatting base method by Kerbl et al. [5] (2023) [S1]. It revisits the problem of photometric stereo, originally addressed by Woodham [12] (1980), using modern differentiable rendering techniques [S1]. The approach was developed to leverage the efficiency and explicit representation of 3D Gaussian Splatting for inverse rendering tasks [S1][S2, unofficial].

## Sources
[S1] Photometric Stereo Using Gaussian Splatting And Inverse Rendering, https://hal.science/hal-05553367v1, fetched 2026-10-26
[S2] Stéréophotométrie par Gaussian Splatting et rendu inverse, https://hal.science/hal-05151330v1, fetched 2026-10-26 (unofficial)
[S3] Geometry-Grounded Gaussian Splatting, https://arxiv.org/abs/2601.17835v2, fetched 2026-10-26 (unofficial)
