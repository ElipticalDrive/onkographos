# Neural BRDF and Geometry Decoding

Also called: Appearance Feature Decoding

## Overview
This method decodes scene appearance features into physically plausible BRDF values, surface normals, and RGB colors using distinct Multi-Layer Perceptrons (MLPs) [S1, unofficial]. It extends the standard NeRF color prediction by explicitly modeling geometric properties and view-dependent reflectance, allowing for accurate relighting and normal estimation [S1, unofficial]. It differs from [MLPRender_Fea](2414-mlprender-fea.md) by outputting normals and BRDF parameters in addition to color, and from [NeRF with Per-Image Embeddings](2484-nerf-with-per-image-embeddings.md) by using explicit geometric decoders rather than just per-image latent codes.

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $a(x)$: Appearance feature vector at point $x$, with length $L$ [S1, unofficial].
    *   $x$: 3D coordinate of the surface point [S1, unofficial].
    *   $d$: View direction vector [S1, unofficial].
    *   $p^*$: Optional physical parameters (e.g., albedo, roughness) [S1, unofficial].
*   **Outputs:**
    *   $rgb$: 3-channel RGB color value [S1, unofficial].
    *   $nval$: 3-channel surface normal vector [S1, unofficial].
    *   $bval$: 3-channel BRDF value [S1, unofficial].

**Data Structures:**
The appearance feature $a(x)$ is a dense vector of length $L$. The MLPs are standard feedforward networks. The specific internal architecture (layer widths, activation functions) is not detailed in the primary source for the BRDF/Normal decoders, though the color decoder $C_{fuc}$ and normal decoder $n_{fuc}$ are described as MLPs [S1, unofficial].

**Numbered Steps:**
1.  **Feature Extraction:** Compute or retrieve the appearance feature vector $a(x)$ for the 3D point $x$ [S1, unofficial].
2.  **Color Decoding:** Pass $a(x)$, $x$, and $d$ into the color MLP $C_{fuc}$ to produce the RGB color [S1, unofficial].
    $$ C_{fuc}: \{ a(x), x, d \} \rightarrow rgb $$
3.  **Normal Decoding:** Pass $a(x)$ and $x$ into the normal MLP $n_{fuc}$ to produce the surface normal [S1, unofficial].
    $$ n_{fuc}: \{ a(x), x \} \rightarrow nval $$
4.  **BRDF Decoding:** Pass $a(x)$, $x$, $d$, and optional parameters $p^*$ into the BRDF MLP $B_{dec}$ to produce the BRDF value [S1, unofficial].
    $$ B_{dec}: \{ a(x), x, d, p^* \} \rightarrow bval $$

**Formulas:**
*   Appearance Feature: $a(x) = F[1, L]$ [S1, unofficial].
*   Color Function: $C_{fuc}: \{ a(x), x, d \} \rightarrow rgb$ [S1, unofficial].
*   Normal Function: $n_{fuc}: \{ a(x), x \} \rightarrow nval$ [S1, unofficial].
*   BRDF Function: $B_{dec}: \{ a(x), x, d, p^* \} \rightarrow bval$ [S1, unofficial].

**Parameters:**
*   $L$: Length of the appearance feature vector. Value not given in the sources.
*   $p^*$: Optional physical parameters such as albedo and roughness. Value not given in the sources.

**Edge Cases and Pitfalls:**
*   **Missing Physical Parameters:** The BRDF decoder accepts optional parameters $p^*$. If these are unavailable, the decoder must handle their absence, though the specific fallback mechanism is not described [S1, unofficial].
*   **Normal Normalization:** The output $nval$ is a 3-channel vector. It must be normalized to unit length to represent a valid surface normal, although the source does not explicitly state if the MLP output is pre-normalized [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neural_brdf_geometry_decoding(appearance_feature, point_x, view_direction, physical_params=None):
    """
    Decodes appearance features into RGB, Normal, and BRDF values.
    
    Args:
        appearance_feature (Tensor): Feature vector a(x) of length L.
        point_x (Tensor): 3D coordinate x.
        view_direction (Tensor): View direction d.
        physical_params (Tensor, optional): Optional parameters p* (e.g., albedo, roughness).
        
    Returns:
        rgb (Tensor): 3-channel RGB color.
        nval (Tensor): 3-channel surface normal.
        bval (Tensor): 3-channel BRDF value.
    """
    # Color Decoding: C_fuc takes a(x), x, and d
    # Note: Specific MLP architecture for C_fuc is not detailed in sources [S1, unofficial]
    rgb = color_mlp(appearance_feature, point_x, view_direction)
    
    # Normal Decoding: n_fuc takes a(x) and x
    # Note: Specific MLP architecture for n_fuc is not detailed in sources [S1, unofficial]
    nval = normal_mlp(appearance_feature, point_x)
    
    # BRDF Decoding: B_dec takes a(x), x, d, and optional p*
    # Note: Specific MLP architecture for B_dec is not detailed in sources [S1, unofficial]
    if physical_params is not None:
        bval = brdf_mlp(appearance_feature, point_x, view_direction, physical_params)
    else:
        bval = brdf_mlp(appearance_feature, point_x, view_direction, None)
        
    return rgb, nval, bval

# Placeholder functions representing the undefined MLPs from [S1, unofficial]
def color_mlp(a, x, d):
    # Implementation details of C_fuc MLP not provided in [S1, unofficial]
    pass

def normal_mlp(a, x):
    # Implementation details of n_fuc MLP not provided in [S1, unofficial]
    pass

def brdf_mlp(a, x, d, p_star):
    # Implementation details of B_dec MLP not provided in [S1, unofficial]
    pass
```

## Subtopics

### [TensoRF Feature Decoding Architecture](8181-tensorf-feature-decoding-architecture.md)
The TensoRF Feature Decoding Architecture maps compressed feature tensors to radiance and density. It is distinguished from Neural BRDF and Geometry Decoding by its ablation-tested linear layer configurations optimized for speed and PSNR, rather than explicit BRDF parameterization.

### [Neural Surface Rendering](9960-neural-surface-rendering.md)
Neural Surface Rendering renders explicit meshes using a neural shader to approximate PBR. It builds on Neural BRDF and Geometry Decoding by explicitly using intersection points, surface normals, and view directions as primary inputs for the appearance model [G].

### [Neural SDF and Material MLPs](10268-neural-sdf-and-material-mlps.md)
This method defines separate MLPs for SDF and material properties (diffuse/specular albedo, roughness) within the IRON pipeline. It differs from TensoRF Feature Decoding Architecture by using explicit separate MLPs rather than tensor-decoded features, and from Neural Surface Rendering by focusing on architectural hyperparameters for inverse rendering.

### [Reflectance NeRF](14040-reflectance-nerf.md)
Reflectance NeRF predicts per-channel reflectance modification factors conditioned on position, view, light direction, and base albedo. It builds on Neural BRDF and Geometry Decoding by providing a specific implementation for decoding appearance features into reflectance multipliers.

### Volumetric Appearance Architectures
This variant models RGB and transparency using two identical neural network architectures (128-128-64 MLPs) with GELU activations [S2]. Inputs include trainable positional encoding of the 3D point, spherical harmonics encoding (degree 3) of the view direction, the SDF normal vector, and a geometric feature vector $z$ predicted by k-SDF [S2]. Normals are computed as normalized gradients of the SDFs using finite differences with $\epsilon = 10^{-4}$ [S2].

## Operating characteristics
Time and memory complexity are determined by the number of MLPs (three distinct networks: color, normal, BRDF) and the length of the appearance feature vector $L$ [S1, unofficial]. The method requires storing and decoding three separate outputs per point, increasing computational load compared to single-output color decoders [S1, unofficial]. Specific performance metrics are not given in the sources.

## Background
Originating from the STensoRRF paper, this method was proposed to accurately relight radiation fields by approximating incident light fields, including direct and indirect illumination [S1, unofficial]. It addresses the limitation of standard NeRF by providing explicit geometric and material properties alongside color [S1, unofficial].

## Sources
[S1] STensoRRF:Tensorial able Relighting Radiance Field for Single-Tensor Scene Representation, https://doi.org/10.20944/preprints202401.1085.v1, fetched 2026-10-30 (unofficial)
[S2] Volumetric Surfaces: Representing Fuzzy Geometries with Layered Meshes, https://doi.org/10.1109/cvpr52734.2025.01991, fetched 2026-10-26
