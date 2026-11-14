# Color and Appearance Modeling

Also called: Appearance conditioning, View-dependent color modeling, Illumination-aware rendering

## Overview
Color and Appearance Modeling in 3D Gaussian Splatting extends the standard radiance representation to handle complex lighting, view-dependent effects, and environmental variations that static Spherical Harmonics (SH) coefficients cannot capture. It solves the problem of inaccurate rendering under varying illumination, specular highlights, and transient objects in unstructured image collections. It differs from standard 3DGS by decoupling geometry from appearance or modeling local illumination fields, and from sibling methods like [Continuous Attenuation Model](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md) by focusing on radiance prediction rather than density/opacity physics.

## Shared principles
All methods in this family optimize Gaussian attributes to predict outgoing radiance $\mathbf{c}$ or modulate base appearance parameters. The core mechanism involves replacing or augmenting the static SH color $\mathbf{c}_{SH}$ with a learned function $f$ that takes Gaussian position, view direction, and potentially global or local context as input. The rendering equation remains the standard alpha-blending compositing:
$$ \hat{\mathbf{C}} = \sum_{i \in N} T_i g_i(\mathbf{u} | \boldsymbol{\mu}', \boldsymbol{\Sigma}') \alpha_i \mathbf{c}_i $$
where $\mathbf{c}_i$ is the predicted color for Gaussian $i$. The optimization minimizes a loss $\mathcal{L}$ between rendered $\hat{\mathbf{C}}$ and ground truth $\mathbf{C}_{gt}$, typically combining L1 and D-SSIM terms.

## Subtopics

### [3iGS (Factorised Tensorial Illumination for 3D Gaussian Splatting)](13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md)
3iGS models outgoing radiance as a function of a continuous local illumination field and individual Gaussian BRDF attributes, rather than optimizing outgoing radiance in isolation [13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md]. It differs from [SWAG](14315-swag-splatting-in-the-wild-images-with-appearance-conditione.md) by using a factorized tensor grid for local illumination rather than appearance-conditioned Gaussians for global lighting changes, and from [GaussianShader](14315-swag-splatting-in-the-wild-images-with-appearance-conditione.md) by using grid-based tensors instead of global environment maps [13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md]. Choose this when accurate specular highlights and local lighting variations are required.

### [SWAG (Splatting in the Wild images with Appearance-conditioned Gaussians)](14315-swag-splatting-in-the-wild-images-with-appearance-conditione.md)
SWAG handles unstructured, in-the-wild image collections by modeling appearance variations and handling transient objects without supervision [14315-swag-splatting-in-the-wild-images-with-appearance-conditione.md]. It differs from standard 3DGS by decoupling static geometry from dynamic appearance and transient occlusions, using per-image embeddings and an MLP to predict color and opacity variations [14315-swag-splatting-in-the-wild-images-with-appearance-conditione.md]. Unlike [3iGS](13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md) which focuses on factorized tensorial illumination, SWAG explicitly isolates transient objects to prevent them from causing "floaters" or blurry elongated Gaussians in the static reconstruction [14315-swag-splatting-in-the-wild-images-with-appearance-conditione.md]. Choose this for in-the-wild datasets with lighting changes and moving objects.

### Compact View-dependent Color
A method to represent view-dependent color using a grid-based neural field instead of storing Spherical Harmonics (SH) coefficients per Gaussian, reducing parameters from 48 to a shared grid [S1].
- **Definition:** Replaces per-Gaussian SH coefficients with a shared hash grid and tiny MLP [S1].
- **Step:** Contract unbounded positions $p \in \mathbb{R}^3$ to a bounded range using the contraction function from mip-NeRF 360 [S1]. Input positions into hash grids, then feed the resulting feature and view direction into the tiny MLP [S1]. Use 0-degree components of SH (same channels as RGB, not view-dependent) and convert to RGB colors [S1].
- **Formula:** Contraction function: $\text{contract}(p_n) = p_n$ if $\|p_n\| \le 1$, else $\frac{p_n}{\|p_n\|} (2 - \|p_n\|)$ [S1]. View-dependent color $c_n(d)$ is computed as: $c_n(d) = f(\text{contract}(p_n), d; \theta)$, where $f$ is a tiny MLP with parameters $\theta$ [S1].
- **Comparison:** Original 3DGS uses 48 parameters per Gaussian for SH (max 3 degrees) to model view-dependent color; this method uses a shared grid-based neural field [S1].

## Background
Originating from extensions to the original 3D Gaussian Splatting paper [Kerbl et al., 2023], these methods address limitations in handling specularities, dynamic lighting, and unstructured data. 3iGS introduces tensorial illumination for physically-based rendering aspects [13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md]. SWAG targets the "in-the-wild" scenario with transient objects [14315-swag-splatting-in-the-wild-images-with-appearance-conditione.md]. Compact View-dependent Color reduces memory footprint by replacing SH with neural fields [S1].

## Sources
[S1] Compact 3D Gaussian Representation for Radiance Field, https://doi.org/10.1109/cvpr52733.2024.02052, fetched 2026-10-26
