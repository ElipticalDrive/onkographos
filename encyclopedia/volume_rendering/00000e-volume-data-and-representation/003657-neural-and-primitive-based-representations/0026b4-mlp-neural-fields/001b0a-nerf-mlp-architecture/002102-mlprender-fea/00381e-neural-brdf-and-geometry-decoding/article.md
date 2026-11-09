# Neural BRDF and Geometry Decoding

Also called: Appearance Feature Decoding

## Overview
This method decodes scene appearance features into explicit BRDF values, surface normals, and RGB colors using distinct neural networks [S1, unofficial]. It solves the problem of separating geometric and material properties from a shared feature representation, allowing for physically plausible relighting and view-dependent shading [S1, unofficial]. It is distinguished from its parent, [MLPRender_Fea](2414-mlprender-fea.md), by explicitly outputting normals and BRDF parameters rather than just RGB, and from [Volume Radiance Network](2576-volume-radiance-network.md) by its structured decomposition of appearance into geometric and material components [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Appearance feature vector $a(x)$, 3D coordinate $x$, view direction $d$, and optional physical parameters $p^*$ (e.g., albedo, roughness) [S1, unofficial].
*   **Outputs:** RGB color $rgb$, surface normal $nval$, and BRDF value $bval$ [S1, unofficial].

**Data Structures:**
*   $a(x)$: A feature vector of length $L$, derived from the scene feature vector $F$ [S1, unofficial].
*   $x$: 3D coordinate $(x, y, z)$ [S1, unofficial].
*   $d$: View direction vector [S1, unofficial].
*   $p^*$: Optional scalar or vector parameters for physical material properties [S1, unofficial].

**Formulas:**
The method employs three distinct functions:
1.  **BRDF Decoding:** $B_{dec}: \{ a(x), x, d, p^* \} \rightarrow bval$ [S1, unofficial].
2.  **Color Decoding:** $C_{fuc}: \{ a(x), x, d \} \rightarrow rgb$ [S1, unofficial].
3.  **Normal Decoding:** $n_{fuc}: \{ a(x), x \} \rightarrow nval$ [S1, unofficial].

Where $bval$ is a 3-channel BRDF value, $rgb$ is the final color, and $nval$ is a 3-channel surface normal [S1, unofficial].

**Steps:**
1.  Extract the appearance feature vector $a(x)$ from the scene feature representation [S1, unofficial].
2.  Pass $a(x)$, $x$, $d$, and $p^*$ into the BRDF decoder $B_{dec}$ to obtain $bval$ [S1, unofficial].
3.  Pass $a(x)$, $x$, and $d$ into the color MLP $C_{fuc}$ to obtain $rgb$ [S1, unofficial].
4.  Pass $a(x)$ and $x$ into the normal MLP $n_{fuc}$ to obtain $nval$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neural_brdf_geometry_decoding(a_x, x, d, p_star=None):
    """
    Decodes appearance features into BRDF, color, and normal.
    
    Args:
        a_x: Appearance feature vector of length L.
        x: 3D coordinate.
        d: View direction vector.
        p_star: Optional physical parameters (albedo, roughness).
        
    Returns:
        bval: 3-channel BRDF value.
        rgb: 3-channel RGB color.
        nval: 3-channel surface normal.
    """
    # B_dec: BRDF decoding
    # Inputs: a(x), x, d, p*
    # Output: bval (3-channel)
    bval = B_dec(a_x, x, d, p_star)
    
    # C_fuc: Color decoding
    # Inputs: a(x), x, d
    # Output: rgb (3-channel)
    rgb = C_fuc(a_x, x, d)
    
    # n_fuc: Normal decoding
    # Inputs: a(x), x
    # Output: nval (3-channel)
    nval = n_fuc(a_x, x)
    
    return bval, rgb, nval
```

**Parameters:**
*   $L$: Length of the appearance feature vector $a(x)$ [S1, unofficial].
*   $p^*$: Optional physical parameters such as albedo and roughness [S1, unofficial].

**Edge Cases and Pitfalls:**
*   The color decoder $C_{fuc}$ explicitly requires the view direction $d$, while the normal decoder $n_{fuc}$ does not [S1, unofficial].
*   The BRDF decoder $B_{dec}$ is the only function that accepts optional physical parameters $p^*$ [S1, unofficial].
*   Precision issues may arise if $a(x)$ is not normalized or if $p^*$ is provided inconsistently across training samples [G].

## Subtopics

### [TensoRF Feature Decoding Architecture](8181-tensorf-feature-decoding-architecture.md)
The TensoRF Feature Decoding Architecture is the multi-layer perceptron (MLP) responsible for mapping compressed feature vectors to final radiance values (color) and volume density (sigma) in the TensoRF framework. It solves the problem of efficiently decoding high-dimensional scene representations into observable pixel values. It is distinguished from its parent, [Neural BRDF and Geometry Decoding](8139-neural-brdf-and-geometry-decoding.md), by its specific focus on standard radiance fields (color and density) rather than explicit BRDF or normal outputs, and from [Neural Surface Rendering](9960-neural-surface-rendering.md) by its volumetric integration approach. The architecture is notably shallow and parameter-efficient, achieving performance comparable to deeper NeRF variants with significantly faster training times.

### [Neural Surface Rendering](9960-neural-surface-rendering.md)
Neural Surface Rendering is a technique that renders explicit surface meshes (or implicit surfaces approximated as meshes) using a neural shader—a coordinate-based Multi-Layer Perceptron (MLP)—to approximate physics-based rendering (PBR). It solves the problem of integrating learned, data-driven appearance models with explicit geometric representations, allowing the network to learn complex shading effects (such as subsurface scattering or view-dependent color shifts) that are difficult to capture with standard analytical BRDFs. Unlike traditional PBR which relies on predefined material parameters, this method learns the mapping from geometric inputs to radiance directly from image observations. It builds on [Neural BRDF and Geometry Decoding](8139-neural-brdf-and-geometry-decoding.md) by explicitly handling the intersection point and normal as primary inputs for the appearance model.

### [Neural SDF and Material MLPs](10268-neural-sdf-and-material-mlps.md)
Defines the neural network architectures for the signed distance function (SDF) and material properties (diffuse albedo, specular albedo, roughness) used in the IRON pipeline.

### Volumetric Appearance Architectures
Two identical neural network architectures (one for RGB, one for transparency) that model view-dependent appearance based on position, view direction, normal, and geometric features [S2]. Both models encode 3D points using trainable positional encoding, followed by an MLP with three layers of 128, 128, and 64 features, respectively. Inputs include encoded position, spherical harmonics encoding (degree 3) of view direction, normal vector, and geometric feature vector [S2].

## Operating characteristics
Time and memory complexity: Not given in the sources.
Measured performance: Not given in the sources.
Scaling: Not given in the sources.
Strengths and limitations: The method allows for explicit control over BRDF and normals, facilitating relighting and material editing [S1, unofficial]. However, it requires more complex network structures (multiple MLPs) compared to single-output decoders [S1, unofficial].
When to use it: When explicit material and geometric parameters are needed for relighting or editing [S1, unofficial].
When not: When only final RGB rendering is required and computational efficiency is prioritized over explicit parameter extraction [G].

## Background
Origin: The method is described in "STensoRRF: Tensorial Relighting Radiance Field for Single-Tensor Scene Representation" [S1, unofficial].
History and development: It extends standard neural radiance fields by decomposing the feature decoding process into separate networks for color, normal, and BRDF, enabling more physically based scene manipulation [S1, unofficial].
Where it is used: In relighting radiance fields and neural surface rendering applications [S1, unofficial].

## Sources
[S1] STensoRRF:Tensorial able Relighting Radiance Field for Single-Tensor Scene Representation, https://doi.org/10.20944/preprints202401.1085.v1, fetched 2026-10-30 (unofficial)
[S2] Volumetric Surfaces: Representing Fuzzy Geometries with Layered Meshes, https://doi.org/10.1109/cvpr52734.2025.01991, fetched 2026-10-26
