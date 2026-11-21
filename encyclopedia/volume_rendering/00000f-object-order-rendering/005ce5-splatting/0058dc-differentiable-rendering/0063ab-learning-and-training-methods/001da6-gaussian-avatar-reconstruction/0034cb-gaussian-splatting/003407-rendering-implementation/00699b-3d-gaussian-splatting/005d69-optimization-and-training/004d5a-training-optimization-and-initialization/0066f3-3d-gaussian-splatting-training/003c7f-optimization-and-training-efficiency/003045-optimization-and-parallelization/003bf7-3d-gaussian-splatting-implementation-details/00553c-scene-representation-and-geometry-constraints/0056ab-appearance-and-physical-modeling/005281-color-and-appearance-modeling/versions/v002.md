# Color and Appearance Modeling

Also called: View-dependent color modeling, Appearance-conditioned rendering, Neural shading for Gaussians

## Overview
Color and Appearance Modeling in 3D Gaussian Splatting extends the standard radiance representation to handle complex lighting, view-dependent effects, and photometric variations that static Spherical Harmonics (SH) coefficients cannot capture. While standard 3DGS optimizes per-Gaussian SH coefficients for outgoing radiance, this family of methods introduces external conditioning mechanisms—such as neural fields, illumination grids, or image embeddings—to disentangle geometry from appearance. This approach solves the problem of blurry or inaccurate specular highlights and transient artifacts in unstructured image collections, setting it apart from siblings by replacing or augmenting the fixed SH basis with learnable, context-aware functions.

## Shared principles
All methods in this family share the goal of computing the outgoing radiance $\mathbf{c}_i$ for a Gaussian $i$ as a function of its position, viewing direction, and potentially external context, rather than a direct lookup from optimized SH coefficients.

1.  **Positional Encoding:** Unbounded 3D positions $\mathbf{p} \in \mathbb{R}^3$ are often contracted to a bounded range to improve neural field stability. The contraction function from mip-NeRF 360 is used [S1]:
    $$
    \text{contract}(\mathbf{p}_n) = 
    \begin{cases} 
    \mathbf{p}_n & \text{if } \|\mathbf{p}_n\| \le 1 \\
    \frac{\mathbf{p}_n}{\|\mathbf{p}_n\|} (2 - \|\mathbf{p}_n\|) & \text{if } \|\mathbf{p}_n\| > 1
    \end{cases}
    $$
2.  **Neural Field Mapping:** A neural network $f_\theta$ (typically a small MLP) maps encoded spatial features and view directions to color or appearance parameters. The general form is:
    $$
    \mathbf{c}_n(\mathbf{d}) = f_\theta(\text{feature}(\mathbf{p}_n), \mathbf{d})
    $$
    where $\mathbf{d}$ is the view direction [S1].
3.  **Modulation:** Appearance is modulated by combining a base color (often from 0-degree SH or a learned base) with a variation term predicted by the neural field or MLP [S1].

## Subtopics

### [3iGS (Factorised Tensorial Illumination for 3D Gaussian Splatting)](13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md)
3iGS models outgoing radiance as a function of a continuous local illumination field and individual Gaussian BRDF attributes, solving the problem of poor specular view-dependent effects in standard 3DGS. It uses a compact, factorized tensor illumination grid and a neural shading network. Choose this for scenes with reflective surfaces requiring realistic specular highlights. It differs from SWAG by using explicit illumination grids rather than per-image latent embeddings.

### [SWAG (Splatting in the Wild images with Appearance-conditioned Gaussians)](14315-swag-splatting-in-the-wild-images-with-appearance-conditione.md)
SWAG reconstructs 3D scenes from unstructured, in-the-wild image collections by disentangling static geometry from dynamic appearance and transient objects. It uses a latent image embedding and an MLP to modulate color and opacity, addressing photometric variations and transient occluders (e.g., tourists, clouds). Choose this for in-the-wild datasets with lighting changes. It differs from 3iGS by using per-image conditioning rather than a global illumination field.

### Compact View-dependent Color
A method to represent view-dependent color using a grid-based neural field instead of storing Spherical Harmonics (SH) coefficients per Gaussian, reducing parameters from 48 to a shared grid [S1].
*   **Definition:** Replaces per-Gaussian SH (48 params) with a shared Hash Grid + MLP [S1].
*   **Step:** Contract unbounded positions $\mathbf{p} \in \mathbb{R}^3$ to a bounded range using the contraction function from mip-NeRF 360. Input positions into hash grids, then feed the resulting feature and view direction into the tiny MLP. Use 0-degree components of SH (same channels as RGB, not view-dependent) and convert to RGB colors [S1].
*   **Formula:** Contraction function: $\text{contract}(\mathbf{p}_n) = \mathbf{p}_n$ if $\|\mathbf{p}_n\| \le 1$, else $\frac{\mathbf{p}_n}{\|\mathbf{p}_n\|} (2 - \|\mathbf{p}_n\|)$. View-dependent color $\mathbf{c}_n(\mathbf{d})$ is computed as: $\mathbf{c}_n(\mathbf{d}) = f(\text{contract}(\mathbf{p}_n), \mathbf{d}; \theta)$, where $f$ is a tiny MLP with parameters $\theta$ [S1].
*   **Comparison:** Original 3DGS uses 48 parameters per Gaussian for SH (max 3 degrees) to model view-dependent color; this method uses a shared grid-based neural field [S1].

## Background
Standard 3D Gaussian Splatting [Parent](9128-3d-gaussian-splatting-implementation-details.md) uses Spherical Harmonics (SH) to represent view-dependent color, which is efficient but limited in modeling complex specularities and lighting variations. The need for advanced appearance modeling arose from the limitations of SH in handling high-frequency reflections and unstructured data. 3iGS introduced factorized tensor illumination grids to model lighting explicitly [S1]. SWAG addressed the "in-the-wild" problem by introducing per-image appearance embeddings to handle transient objects and lighting changes [S1]. Compact View-dependent Color proposed replacing the heavy SH storage with a shared neural field to reduce memory usage [S1].

## Sources
[S1] Compact 3D Gaussian Representation for Radiance Field, https://doi.org/10.1109/cvpr52733.2024.02052, fetched 2026-10-26
