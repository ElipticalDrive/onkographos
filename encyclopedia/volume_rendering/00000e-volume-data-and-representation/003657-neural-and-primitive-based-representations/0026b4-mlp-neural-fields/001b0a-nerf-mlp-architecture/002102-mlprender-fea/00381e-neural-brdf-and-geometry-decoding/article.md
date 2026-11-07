# Neural BRDF and Geometry Decoding

Also called: Appearance Feature Decoding

## Overview
This method decodes appearance features $a(x)$ into Bidirectional Reflectance Distribution Function (BRDF) values, surface normals, and RGB colors using distinct neural networks [S1, unofficial]. It builds on [MLPRender_Fea](2414-mlprender-fea.md) by separating the decoding of geometric properties (normals, BRDF) from photometric properties (color) into specialized MLPs, allowing for explicit physical parameterization [S1, unofficial]. Unlike [NeRF with Per-Image Embeddings](2484-nerf-with-per-image-embeddings.md) which adjusts global appearance, or [Linear Feature Decoder with Learned Color Basis](3352-linear-feature-decoder-with-learned-color-basis.md) which uses linear projections, this approach employs non-linear MLPs to map features to physical quantities [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $a(x)$: Appearance feature vector at point $x$, derived from the scene feature vector $F$ [S1, unofficial].
    *   $x$: 3D coordinate of the surface point.
    *   $d$: View direction vector.
    *   $p^*$: Optional physical parameters (e.g., albedo, roughness) [S1, unofficial].
*   **Outputs:**
    *   $bval$: 3-channel BRDF value.
    *   $rgb$: 3-channel RGB color value.
    *   $nval$: 3-channel surface normal vector.

**Data Structures:**
The appearance feature $a(x)$ is a vector of length $L$ [S1, unofficial]. The MLPs are standard feedforward networks with linear layers and ReLU activations.

**Formulas:**
1.  BRDF Decoding:
    $$ B_{dec}: \{ a(x), x, d, p^* \} \rightarrow bval $$
    [S1, unofficial]
2.  Color Decoding:
    $$ C_{fuc}: \{ a(x), x, d \} \rightarrow rgb $$
    [S1, unofficial]
3.  Normal Decoding:
    $$ n_{fuc}: \{ a(x), x \} \rightarrow nval $$
    [S1, unofficial]

**Parameters:**
*   $p^*$: Optional. If available, included in the BRDF decoder input [S1, unofficial].
*   $mlpC$: Input dimension for the color MLP, corresponding to the feature dimension [S2, unofficial].
*   $featureC$: Hidden dimension for the color MLP [S2, unofficial].

**Steps:**
1.  Extract appearance feature $a(x)$ from the scene representation at point $x$ [S1, unofficial].
2.  **Decode BRDF:** Pass $a(x)$, $x$, $d$, and optional $p^*$ into the BRDF MLP ($B_{dec}$) to output $bval$ [S1, unofficial].
3.  **Decode Color:** Pass $a(x)$, $x$, and $d$ into the Color MLP ($C_{fuc}$) to output $rgb$ [S1, unofficial].
4.  **Decode Normal:** Pass $a(x)$ and $x$ into the Normal MLP ($n_{fuc}$) to output $nval$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decode_brdf(a, x, d, p_star=None):
    """
    Decodes BRDF values from appearance features.
    Args:
        a: Appearance feature vector [L]
        x: 3D coordinate [3]
        d: View direction [3]
        p_star: Optional physical parameters
    Returns:
        bval: BRDF values [3]
    """
    # Input concatenation: a, x, d, and optionally p_star
    input_vec = concatenate([a, x, d])
    if p_star is not None:
        input_vec = concatenate([input_vec, p_star])
    
    # MLP processing (specific architecture not detailed in S1, assumed standard MLP)
    # [G] Standard MLP implementation: Linear -> ReLU -> Linear -> ... -> Linear
    # The exact layer sizes are not specified in S1 for B_dec.
    bval = mlp_brdf(input_vec)
    return bval

def decode_color(a, x, d):
    """
    Decodes RGB color from appearance features.
    Args:
        a: Appearance feature vector [L]
        x: 3D coordinate [3]
        d: View direction [3]
    Returns:
        rgb: RGB color values [3]
    """
    input_vec = concatenate([a, x, d])
    rgb = mlp_color(input_vec)
    return rgb

def decode_normal(a, x):
    """
    Decodes surface normal from appearance features.
    Args:
        a: Appearance feature vector [L]
        x: 3D coordinate [3]
    Returns:
        nval: Surface normal vector [3]
    """
    input_vec = concatenate([a, x])
    nval = mlp_normal(input_vec)
    return nval
```

**Edge Cases and Pitfalls:**
*   The BRDF decoder requires view direction $d$ and optional physical parameters $p^*$, while the normal decoder does not use $d$ [S1, unofficial].
*   The sources do not specify the exact architecture (layer sizes, number of layers) for $B_{dec}$ and $n_{fuc}$, only for the color decoder in the TensoRF variant [S1, unofficial][S2, unofficial].

## Subtopics

### TensoRF Feature Decoding Architecture
This variant specifies the architecture for the color MLP ($C_{fuc}$) used in TensoRF [S2, unofficial].
*   **Default Architecture:** 3 linear layers with ReLU activations [S2, unofficial].
    *   Layer 1: Linear(in $mlpC$, $featureC$), ReLU.
    *   Layer 2: Linear($featureC$, $featureC$), ReLU.
    *   Layer 3: Linear($featureC$, 3) [S2, unofficial].
*   **Variants Tested:**
    1.  **Convolutional Layers:** Replaced linear layers with 1D convolutional layers with small kernel sizes to reduce parameters and speed up training [S2, unofficial].
    2.  **Residual from Initial Input:** Added residual connections from the first layer to each subsequent layer by concatenating a part of the initial input to each layer [S2, unofficial].
    3.  **Residual from Alternate Layers:** Added residual connections from 2 layers prior to each layer to preserve information while tuning parameters [S2, unofficial].
    4.  **Deeper Network:** Added 3 linear layers along with residuals from sigma layers to increase parameters for better color output [S2, unofficial].
*   **Comparison:** Adding/removing linear layers or changing depths (e.g., 5 layers with sizes 64, 32, 16, 8) did not make significant improvements in PSNR [S2, unofficial]. The default 3-layer architecture achieved comparable PSNR to deeper variants but trained faster [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** In the TensoRF variant, the default 3-layer decoder achieved a Train PSNR of 31.11 and Test PSNR of 31.297 on the Lego scene (downsampled by factor 4, 2500 iterations) [S2, unofficial]. This is comparable to NeRF's ~31 PSNR but trained in less than 10 minutes [S2, unofficial].
*   **Strengths:** Allows explicit decoding of normals and BRDF, enabling relighting and physical parameterization [S1, unofficial]. The default 3-layer architecture is efficient [S2, unofficial].
*   **Limitations:** The specific architecture for BRDF and Normal decoders is not detailed in the primary source [S1, unofficial]. Performance gains from architectural changes (depth, residuals) are marginal [S2, unofficial].
*   **When to use:** When explicit geometric (normal) and material (BRDF) information is required for relighting or physical simulation [S1, unofficial].
*   **When not to use:** When computational resources are extremely limited and only color prediction is needed, as the separate MLPs add overhead compared to a single color-only decoder.

## Background
The method originates from STensoRRF, a tensorial relighting radiance field for single-tensor scene representation [S1, unofficial]. It was proposed to accurately relight radiation fields by approximating incident light fields, considering direct light from sources, direct light received by surface points, and indirect light reflected from other surfaces [S1, unofficial]. The approach imitates incident light simulation methods using spherical Gaussians for global illumination [S1, unofficial].

## Sources
[S1] STensoRRF:Tensorial able Relighting Radiance Field for Single-Tensor Scene Representation, https://doi.org/10.20944/preprints202401.1085.v1, fetched 2026-10-30 (unofficial)
[S2] Low-Cost Neural Radiance Fields, https://doi.org/10.48550/arxiv.2605.09312, fetched 2026-10-26 (unofficial)
