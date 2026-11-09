# Ref-NeRF

Also called: Structured View-Dependent Appearance for Neural Radiance Fields

## Overview
Ref-NeRF is a neural radiance field method that replaces standard NeRF’s parameterization of view-dependent outgoing radiance with a representation of reflected radiance, structured using spatially-varying scene properties to improve specular reflection realism [S1]. It sets itself apart from sibling methods by explicitly decomposing radiance into diffuse and specular components and reparameterizing the directional input using reflection vectors, rather than relying on raw viewing directions or spherical harmonics [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D spatial coordinates $\mathbf{x}$, viewing direction $\mathbf{d}$, and surface normal $\mathbf{n}$ (estimated from density).
*   **Outputs:** Volume density $\sigma$ and outgoing radiance $L_o$, decomposed into diffuse $L_d$ and specular $L_s$ components.

**Data Structures:**
The model uses two primary MLPs: a spatial MLP and a directional MLP.
1.  **Spatial MLP:** Takes $\mathbf{x}$ and outputs a feature vector $\mathbf{c}$, volume density $\sigma$, and an intermediate feature for the normal estimation.
2.  **Directional MLP:** Takes the reflection vector $\mathbf{r}$ and feature $\mathbf{c}$ to output the specular radiance. The diffuse radiance is typically computed as a function of albedo and diffuse lighting, often independent of the viewing direction in the simplest formulation, or via a separate diffuse directional branch.

**Numbered Steps:**

1.  **Normal Estimation and Regularization:**
    Standard NeRF geometry is "foggy," leading to noisy normals unsuitable for reflection calculations [S1]. Ref-NeRF applies a volume density regularizer to concentrate density around surfaces. This ensures that the gradient of the density field $\nabla \sigma$ provides a sharp, accurate normal vector $\mathbf{n} = \frac{\nabla \sigma}{\|\nabla \sigma\|}$ [S1].

2.  **Reflection Vector Computation:**
    Instead of using the viewing direction $\mathbf{d}$ as input to the directional MLP, Ref-NeRF computes the reflection vector $\mathbf{r}$ of $\mathbf{d}$ about the local normal $\mathbf{n}$:
    $$ \mathbf{r} = 2(\mathbf{d} \cdot \mathbf{n})\mathbf{n} - \mathbf{d} $$
    This reparameterization ensures that the reflected radiance function is constant across the scene for distant illumination (ignoring occlusions), making it smoother and easier for the neural network to interpolate [S1].

3.  **Integrated Directional Encoding (IDE):**
    To handle varying materials and textures smoothly, Ref-NeRF employs Integrated Directional Encoding. This technique encodes the reflection vector $\mathbf{r}$ (and potentially the viewing direction) using a specific basis (e.g., spherical harmonics or positional encoding) before feeding it into the directional MLP. This allows the model to capture high-frequency specular details while maintaining smoothness in the underlying radiance function [S1].

4.  **Radiance Decomposition:**
    The outgoing radiance is split into explicit diffuse and specular components:
    $$ L_o = L_d + L_s $$
    *   **Diffuse ($L_d$):** Computed based on material albedo and diffuse lighting. It is generally view-independent or weakly view-dependent.
    *   **Specular ($L_s$):** Computed by the directional MLP using the encoded reflection vector $\mathbf{r}$ and spatial features $\mathbf{c}$. This separation allows the specular component to remain smooth despite variations in material and texture [S1].

5.  **Volume Rendering:**
    Standard volume rendering integration is performed along the ray, accumulating the density $\sigma$ and the decomposed radiance $L_o$ to produce the final pixel color [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import torch

def compute_normal(density_field, x, epsilon=1e-3):
    """
    Estimate normal from density field using central differences.
    density_field: callable returning density at x
    x: tensor of shape (3,)
    """
    # Compute gradients using finite differences
    grad_x = torch.zeros_like(x)
    for i in range(3):
        x_plus = x.clone()
        x_minus = x.clone()
        x_plus[i] += epsilon
        x_minus[i] -= epsilon
        grad_x[i] = (density_field(x_plus) - density_field(x_minus)) / (2 * epsilon)
    
    # Normalize to get unit normal
    n = grad_x / (torch.norm(grad_x) + 1e-8)
    return n

def reflection_vector(d, n):
    """
    Compute reflection vector r of viewing direction d about normal n.
    d: tensor of shape (3,)
    n: tensor of shape (3,)
    """
    dot_product = torch.dot(d, n)
    r = 2 * dot_product * n - d
    return r

def integrated_directional_encoding(r, encoding_fn):
    """
    Apply Integrated Directional Encoding to the reflection vector.
    r: tensor of shape (3,)
    encoding_fn: function that encodes a vector (e.g., spherical harmonics)
    """
    # Placeholder for specific IDE implementation details not fully specified in source
    # Typically involves projecting r onto a basis
    encoded_r = encoding_fn(r)
    return encoded_r

def ref_nerf_forward(x, d, spatial_mlp, directional_mlp, density_field, encoding_fn):
    """
    Forward pass for Ref-NeRF.
    """
    # 1. Get spatial features and density
    spatial_features, sigma = spatial_mlp(x)
    
    # 2. Estimate normal from density for reflection calculation
    n = compute_normal(density_field, x)
    
    # 3. Compute reflection vector
    r = reflection_vector(d, n)
    
    # 4. Encode reflection vector
    encoded_r = integrated_directional_encoding(r, encoding_fn)
    
    # 5. Compute specular radiance
    # Concatenate spatial features and encoded reflection vector
    directional_input = torch.cat([spatial_features, encoded_r], dim=-1)
    specular_radiance = directional_mlp(directional_input)
    
    # 6. Compute diffuse radiance (simplified as view-independent albedo * diffuse light)
    # In practice, this might involve a separate diffuse MLP or simple albedo lookup
    diffuse_radiance = compute_diffuse_radiance(spatial_features) 
    
    # 7. Combine
    total_radiance = diffuse_radiance + specular_radiance
    
    return sigma, total_radiance
```

**Parameters:**
*   **Regularization Weight:** The weight for the density concentration regularizer is critical. Not given in the sources.
*   **Encoding Basis:** The specific basis used for Integrated Directional Encoding (e.g., number of spherical harmonic bands). Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Foggy Geometry:** If the density regularizer is insufficient, normals will be noisy, leading to incorrect reflection vectors and artifacts [S1].
*   **Degenerate Normals:** If $\nabla \sigma$ is near zero (flat density regions), normal estimation fails. A small epsilon is added for stability [G].

## Operating characteristics
*   **Time Complexity:** Similar to standard NeRF, as it adds normal estimation and reflection vector computation, which are low-cost compared to MLP inference.
*   **Memory Complexity:** Similar to standard NeRF, with minor overhead for storing intermediate features for normal estimation.
*   **Performance:** On the evaluated dataset, Ref-NeRF achieves 35.6dB PSNR and 11.5 degrees Mean Angular Error (MAE) for normals, compared to mip-NeRF's 30.3dB PSNR and 59.5 degrees MAE [S1].
*   **Strengths:** Significantly improves the realism and accuracy of specular reflections; provides an interpretable internal representation useful for scene editing [S1].
*   **Limitations:** Relies heavily on accurate normal estimation, which requires careful regularization of the density field [S1].

## Background
Ref-NeRF was introduced by Dor Verbin, Peter Hedman, Ben Mildenhall, Todd Zickler, Jonathan T. Barron, and Pratul P. Srinivasan from Harvard University and Google Research [S1]. It was published as arXiv:2112.03907v1 on Dec 7, 2021 [S1]. The method addresses the limitation of NeRF-based techniques in accurately capturing glossy surfaces by structuring the view-dependent appearance using spatially-varying scene properties [S1].

## Sources
[S1] Ref-NeRF: Structured View-Dependent Appearance for Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00541, fetched 2026-10-26
