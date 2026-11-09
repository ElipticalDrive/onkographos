# Ref-NeRF

Also called: Structured View-Dependent Appearance for Neural Radiance Fields

## Overview
Ref-NeRF is a neural radiance field method that replaces NeRF’s parameterization of view-dependent outgoing radiance with a representation of reflected radiance, structured using spatially-varying scene properties to improve specular reflection realism [S1]. It sets itself apart from standard NeRF and its sibling [Anisotropic Neural Representation Learning](9475-anisotropic-neural-representation-learning.md) by explicitly decomposing radiance into diffuse and specular components and reparameterizing the directional input using surface normals, allowing for accurate interpolation of glossy appearances [S1].

## How it works
Ref-NeRF builds on the [Appearance and Lighting Decomposition](11061-appearance-and-lighting-decomposition.md) principles. It modifies the directional MLP to predict reflected radiance rather than outgoing radiance directly.

**Inputs and Outputs:**
*   **Inputs:** 3D position $\mathbf{x} \in \mathbb{R}^3$, viewing direction $\mathbf{d} \in \mathbb{R}^3$, and surface normal $\mathbf{n} \in \mathbb{R}^3$.
*   **Outputs:** Volume density $\sigma \in \mathbb{R}$, diffuse color $\mathbf{c}_{diff} \in \mathbb{R}^3$, and specular color $\mathbf{c}_{spec} \in \mathbb{R}^3$.

**Data Structures:**
The model uses two coupled MLPs, $F_\sigma$ and $F_c$, consistent with the parent article.
*   $F_\sigma$: Takes encoded position $\gamma(\mathbf{x})$. Outputs density $\sigma$ and an intermediate feature vector $\mathbf{f}$.
*   $F_c$: Takes encoded position $\gamma(\mathbf{x})$, intermediate feature $\mathbf{f}$, and a reparameterized direction input. Outputs $\mathbf{c}_{diff}$ and $\mathbf{c}_{spec}$.

**Numbered Steps:**

1.  **Positional Encoding:** Encode position $\mathbf{x}$ using $\gamma(\mathbf{x})$ as defined in the parent article [S1][S3].
2.  **Density and Feature Prediction:** Pass $\gamma(\mathbf{x})$ through $F_\sigma$ to obtain density $\sigma$ and feature vector $\mathbf{f}$ [S1].
3.  **Normal Estimation:** Compute the surface normal $\mathbf{n}$ at $\mathbf{x}$ by taking the negative gradient of the density field:
    $$ \mathbf{n} = -\frac{\nabla \sigma(\mathbf{x})}{\|\nabla \sigma(\mathbf{x})\|} $$
    [S1]. Note: NeRF's geometry is naturally "foggy," making $\mathbf{n}$ noisy. A specific regularizer is applied to $\sigma$ to concentrate density around surfaces and stabilize $\mathbf{n}$ [S1].
4.  **Reflection Vector Calculation:** Compute the reflection of the viewing direction $\mathbf{d}$ about the normal $\mathbf{n}$:
    $$ \mathbf{r} = 2(\mathbf{d} \cdot \mathbf{n})\mathbf{n} - \mathbf{d} $$
    [S1]. This $\mathbf{r}$ serves as the directional input to the color network, replacing $\mathbf{d}$ [S1].
5.  **Integrated Directional Encoding (IDE):** Apply Integrated Directional Encoding to $\mathbf{r}$ to create a smooth interpolation kernel for the reflected radiance function. This involves encoding $\mathbf{r}$ with sinusoidal functions, similar to $\gamma(\mathbf{x})$, but tailored for directional interpolation [S1].
6.  **Color Prediction:** Pass $\gamma(\mathbf{x})$, $\mathbf{f}$, and the encoded $\mathbf{r}$ through $F_c$. The network outputs separate diffuse and specular color components $\mathbf{c}_{diff}$ and $\mathbf{c}_{spec}$ [S1].
7.  **Radiance Composition:** The total outgoing radiance $\mathbf{c}$ is the sum of the diffuse and specular components:
    $$ \mathbf{c} = \mathbf{c}_{diff} + \mathbf{c}_{spec} $$
    [S1]. This separation allows the reflected radiance function (dependent on $\mathbf{r}$) to remain smooth despite variations in material and texture [S1].
8.  **Volumetric Rendering:** Render the pixel color $C(\mathbf{r})$ using the standard volume rendering equation with the predicted $\sigma$ and $\mathbf{c}$ [S1][S3].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch

def positional_encoding(x, num_freq_bands):
    """Standard positional encoding for coordinates."""
    freq_bands = 2.0 ** torch.linspace(0, num_freq_bands - 1, num_freq_bands)
    scaled_x = x.unsqueeze(-1) * freq_bands * math.pi
    return torch.cat([torch.sin(scaled_x), torch.cos(scaled_x)], dim=-1)

def integrated_directional_encoding(r, num_freq_bands):
    """
    Integrated Directional Encoding for the reflection vector r.
    Structured to allow smooth interpolation of reflected radiance.
    """
    freq_bands = 2.0 ** torch.linspace(0, num_freq_bands - 1, num_freq_bands)
    scaled_r = r.unsqueeze(-1) * freq_bands * math.pi
    return torch.cat([torch.sin(scaled_r), torch.cos(scaled_r)], dim=-1)

def compute_normal(density_fn, x, epsilon=1e-4):
    """
    Compute surface normal from density gradient.
    Uses finite differences for numerical stability.
    """
    # Compute gradient of density at x
    # Note: In practice, this is often done via automatic differentiation
    # during the training loop. Here we assume a helper that returns grad.
    grad_sigma = torch.autograd.grad(
        density_fn(x).sum(), x, create_graph=True
    )[0]
    normal = -grad_sigma / (torch.norm(grad_sigma) + 1e-8)
    return normal

def reflect_vector(d, n):
    """Reflect viewing direction d about normal n."""
    return 2.0 * torch.dot(d, n) * n - d

def ref_nerf_forward(x, d, density_mlp, color_mlp, num_freq_bands):
    """
    Forward pass for Ref-NeRF.
    
    Args:
        x: 3D position (3,)
        d: Viewing direction (3,)
        density_mlp: MLP for density and features
        color_mlp: MLP for diffuse and specular colors
        num_freq_bands: Number of frequency bands for encoding
        
    Returns:
        sigma: Volume density
        c_diff: Diffuse color
        c_spec: Specular color
    """
    # 1. Encode position
    x_enc = positional_encoding(x, num_freq_bands)
    
    # 2. Get density and feature vector
    # density_mlp outputs (sigma, feature_vector)
    sigma, feature_vec = density_mlp(x_enc)
    
    # 3. Estimate normal
    # In a real implementation, density_fn would be a wrapper around density_mlp
    # that returns scalar sigma for gradient computation.
    normal = compute_normal(lambda px: density_mlp(positional_encoding(px, num_freq_bands))[0], x)
    
    # 4. Compute reflection vector
    r = reflect_vector(d, normal)
    
    # 5. Encode reflection vector using IDE
    r_enc = integrated_directional_encoding(r, num_freq_bands)
    
    # 6. Predict diffuse and specular colors
    # color_mlp takes (x_enc, feature_vec, r_enc)
    # It outputs (c_diff, c_spec)
    c_diff, c_spec = color_mlp(x_enc, feature_vec, r_enc)
    
    return sigma, c_diff, c_spec
```

**Parameters:**
*   `num_freq_bands`: Number of frequency bands for positional and directional encoding. Typical values are not explicitly given in the sources, but standard NeRF uses 10 [S1][S3].
*   Regularizer weight for volume density: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Noisy Normals:** NeRF's geometry is "foggy," leading to noisy normal vectors. This is mitigated by the volume density regularizer [S1].
*   **Degenerate Normals:** If the density gradient is zero (e.g., in empty space), the normal computation may fail. A small epsilon is added to the norm denominator [G].

## Operating characteristics
*   **Time Complexity:** Similar to NeRF, dominated by MLP evaluations per sample. The additional normal computation and reflection vector calculation add negligible overhead [S1].
*   **Memory Complexity:** Similar to NeRF. The model stores the same MLP parameters plus the logic for normal estimation [S1].
*   **Performance:** On the evaluated dataset, Ref-NeRF achieves 35.6dB PSNR and 11.5 degrees Mean Angular Error (MAE) for normals, compared to mip-NeRF's 30.3dB PSNR and 59.5 degrees MAE [S1].
*   **Strengths:** Significantly improves the realism and accuracy of specular reflections. The internal representation is interpretable and useful for scene editing [S1].
*   **Limitations:** Relies on accurate normal estimation, which requires the density regularizer. Performance on non-glossy surfaces may not differ significantly from standard NeRF [S1].

## Background
Ref-NeRF was introduced by Dor Verbin, Peter Hedman, Ben Mildenhall, Todd Zickler, Jonathan T. Barron, and Pratul P. Srinivasan from Harvard University and Google Research. It was published as arXiv:2112.03907v1 on Dec 7, 2021 [S1]. The method addresses the limitation of NeRF-based techniques in accurately capturing glossy surfaces by reparameterizing the view-dependent appearance using reflected radiance [S1].

## Sources
[S1] Ref-NeRF: Structured View-Dependent Appearance for Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00541, fetched 2026-10-26
