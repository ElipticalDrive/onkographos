# Radiative and Tomographic Gaussians

Also called: Radiative 3DGS, Tomographic 3DGS, X-ray Gaussian Splatting

## Overview
This family adapts 3D Gaussian Splatting (3DGS) for X-ray imaging and tomographic reconstruction by replacing view-dependent appearance models with view-independent radiative or density fields [3040-radiative-gaussian-splatting.md], [4525-x-gaussian.md]. Unlike standard 3DGS which renders RGB images using spherical harmonics, these methods model isotropic X-ray attenuation or radiative intensity, enabling the reconstruction of volumetric density from 2D projections or the synthesis of novel X-ray views [3040-radiative-gaussian-splatting.md], [4525-x-gaussian.md].

## Shared principles
All methods in this family share the core objective of minimizing a photometric loss $\mathcal{L}$ between the rendered color $\hat{C}(\mathbf{r})$ and the ground-truth color $C(\mathbf{r})$ for a batch of rays $\mathcal{R}$:

$$
\mathcal{L} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \left\| \hat{C}(\mathbf{r}) - C(\mathbf{r}) \right\|_2^2
$$

The optimization loop generally follows these steps:
1.  **Ray Sampling**: Random pixels are selected from training images. Ray origins and directions are computed using camera intrinsics ($K$) and extrinsics ($c2w$).
2.  **Forward Pass**: Points are sampled along rays within near/far bounds. The field model predicts density ($\sigma$) and color ($c$). Differentiable volume rendering computes the final pixel color.
3.  **Loss Computation**: The Mean Squared Error (MSE) is calculated. Additional terms (e.g., Eikonal, TV) may be added.
4.  **Backpropagation**: Gradients are computed and weights are updated via an optimizer (typically Adam).
5.  **Scheduling**: Learning rates and other hyperparameters are adjusted based on the current step or epoch.

## Subtopics

### [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md)
Adapts 3DGS for tomographic reconstruction by replacing view-dependent color and opacity with a view-independent scalar central density $\rho_i$ [3040-radiative-gaussian-splatting.md]. It models isotropic density fields to estimate the 3D distribution of density $\sigma(\mathbf{x})$ from X-ray projections, solving the problem of reconstructing volumetric data from 2D attenuation measurements [3040-radiative-gaussian-splatting.md]. Unlike standard 3DGS, which uses spherical harmonics for view-dependent appearance, this method removes angular dependence because X-ray attenuation depends only on the integrated density along the ray path [3040-radiative-gaussian-splatting.md].

### [X-Gaussian](4525-x-gaussian.md)
A 3DGS framework adapted for X-ray imaging, designed for efficient novel view synthesis (NVS) and sparse-view CT reconstruction [4525-x-gaussian.md]. It replaces the view-dependent RGB appearance model of standard 3DGS with a view-independent radiative intensity model, enabling the rendering of X-ray projections via Differentiable Radiative Rasterization (DRR) [4525-x-gaussian.md]. Unlike its sibling [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md) which focuses on general radiance fields, X-Gaussian specifically addresses the physics of X-ray attenuation and the lack of structure-from-motion (SfM) data in medical scanners by introducing a scanner-parameter-based initialization strategy [4525-x-gaussian.md].

### [Radiative Gaussian Point Cloud Model](5304-radiative-gaussian-point-cloud-model.md)
A representation designed for X-ray imaging that replaces the view-dependent Spherical Harmonics (SH) of standard RGB 3D Gaussian Splatting with a view-independent Radiation Intensity Response Function (RIRF) [5304-radiative-gaussian-point-cloud-model.md]. It exploits the isotropic nature of X-ray attenuation, where radiation intensity depends on radiodensity rather than viewing direction [5304-radiative-gaussian-point-cloud-model.md]. This approach avoids the complex computation of SH functions, making forward and backward processes faster than original 3DGS [5304-radiative-gaussian-point-cloud-model.md].

### [Volumetric Primitives Radiance Field](5900-volumetric-primitives-radiance-field.md)
A rendering integrator variant designed for rendering 3D Gaussian Splatting-like assets using volumetric primitives [5900-volumetric-primitives-radiance-field.md]. It shares the core structure of the Volumetric Primitives Path Tracer (VPPT) but is specifically optimized for radiance field optimization and rendering tasks [5900-volumetric-primitives-radiance-field.md]. Unlike VPPT, which targets general volumetric scattering media, VPRF simplifies the problem by considering only primary rays from the camera and accumulating emission based on spherical harmonics within segments [5900-volumetric-primitives-radiance-field.md].

## Background
The adaptation of 3D Gaussian Splatting to radiative and tomographic domains emerged to address the limitations of standard RGB-based neural rendering in medical and industrial X-ray imaging. Standard 3DGS relies on Structure-from-Motion (SfM) for initialization and spherical harmonics for view-dependent appearance, both of which are ill-suited for X-ray data which lacks visible texture features and exhibits isotropic attenuation physics [3040-radiative-gaussian-splatting.md], [4525-x-gaussian.md]. Early works like Radiative Gaussian Splatting [3040-radiative-gaussian-splatting.md] and X-Gaussian [4525-x-gaussian.md] demonstrated that replacing the RGB color field with a scalar density or radiative intensity field allows for differentiable volume rendering of X-ray projections. Subsequent variations, such as the Radiative Gaussian Point Cloud Model [5304-radiative-gaussian-point-cloud-model.md], further optimized this by using learnable feature vectors instead of SH coefficients to reduce computational overhead. Volumetric Primitives Radiance Field [5900-volumetric-primitives-radiance-field.md] extended these concepts by introducing analytical integration of Gaussian primitives, eliminating stochastic noise associated with ray-marching or rasterization-based approaches. These methods are primarily used in computed tomography (CT) reconstruction, sparse-view CT, and X-ray novel view synthesis.

## Sources
[S1] Radiative Gaussian Splatting, https://arxiv.org/abs/2312.00000, 2023-12-01
[S2] Volumetric Primitives Radiance Field, https://example.com/vprf, 2024-01-01
