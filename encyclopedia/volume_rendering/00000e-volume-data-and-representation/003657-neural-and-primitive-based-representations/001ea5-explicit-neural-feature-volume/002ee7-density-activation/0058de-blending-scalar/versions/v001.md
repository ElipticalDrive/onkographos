# Blending scalar


## Overview
Blending scalar is a method for computing a view-dependent blending weight $\gamma$ in neural inverse rendering, typically used to modulate surface reflectance or shading cues based on geometry and viewing angle [S1]. It differs from standard density activation methods (which map features to scalar density/opacity) by operating on surface normals and roughness to produce a scalar blending factor rather than a volumetric density [S1]. Unlike Gaussian Parameter Activation or Density Mapping Layers which focus on volumetric or point-cloud density representation, blending scalar specifically addresses the interaction between surface geometry (normal, roughness) and view direction [S1].

## How it works
The method computes a scalar blending value $\gamma$ using a Multi-Layer Perceptron (MLP) that takes geometric and material properties as input [S1].

**Inputs and Outputs:**
*   **Inputs:**
    *   $\|x - x'\|$: Distance between points (or coordinate difference).
    *   $w_i \cdot n$: Dot product of the view direction $w_i$ and surface normal $n$.
    *   $\rho_r$: Neural roughness feature.
*   **Output:**
    *   $\gamma$: A scalar blending weight.

**Data Structures:**
*   **MLP Architecture:** A 4-layer MLP with a hidden width of 128 [S1].
*   **Input Encoding:**
    *   The dot product term $(w_i \cdot n)$ is encoded using 6 frequency bands [S1].
    *   The input location $x$ (if used as part of the broader feature vector context in [S1]) is encoded with 6 frequencies [S1].

**Steps:**
1.  **Encode Inputs:** Apply positional encoding with 6 frequencies to the dot product of the view direction and surface normal ($w_i \cdot n$) [S1].
2.  **Compute Neural Roughness:** Calculate the neural roughness feature $\rho_r$ using a separate 4-layer MLP (width 256) taking input $x$, normal $n$, and geometric features $f_{geo}$ [S1].
3.  **MLP Forward Pass:** Pass the encoded inputs and $\rho_r$ into the 4-layer blending scalar MLP (width 128) [S1].
4.  **Output:** The output of the MLP is the blending scalar $\gamma$ [S1].

**Formulas:**
The neural roughness is defined as:
$$ f_{\Theta_r}(x, n, f_{geo}) = \rho_r $$
The blending scalar is defined as:
$$ \gamma(\|x - x'\|, w_i \cdot n, \rho_r) = \gamma $$

**Parameters:**
*   MLP Layers: 4 [S1].
*   MLP Width: 128 [S1].
*   Encoding Frequencies: 6 [S1].

**Edge Cases and Pitfalls:**
*   The method relies on accurate surface normals $n$ and view directions $w_i$. Degenerate normals or grazing angles may produce unstable dot products if not normalized.
*   The roughness feature $\rho_r$ must be computed consistently with the geometry encoder.

## Subtopics

### Appearance Representation
This variant, described in the context of Implicit Neural Point Clouds (INPC), uses a multi-resolution hash grid to retrieve appearance features for points [S2]. It differs from the standard blending scalar by explicitly modeling view-dependent color via Spherical Harmonics (SH) rather than just a scalar blend [S2].
*   **Feature Retrieval:** Retrieve $M+1$ appearance features from a multi-resolution hash grid for each point. Spherical contraction [Barron et al.] is applied to increase capacity near the scene center [S2].
*   **Opacity Conversion:** The first feature $x$ is converted to opacity $\alpha_h \in [0, 1]$ using:
    $$ \alpha_h = 1 - e^{-e^x} $$
    [S2].
*   **SH Evaluation:** The remaining $M$ features serve as coefficients for Spherical Harmonics (SH) evaluation to produce a view-dependent feature [S2].
*   **Background Model:** A small MLP computes a 4D background feature using the SH-encoded viewing direction [S2].
*   **Parameters:** $M=36$ (SH degree 2, 4D output features) [S2].

## Operating characteristics
*   **Time Complexity:** Dominated by the MLP forward pass (4 layers, width 128) and the encoding of 6 frequencies. This is lightweight compared to full radiance field evaluations.
*   **Memory Complexity:** Low. The MLP weights are fixed. The hash grid in the appearance variant requires memory proportional to the grid size and number of points.
*   **Strengths:** Efficiently captures view-dependent effects without storing explicit textures. Integrates seamlessly with neural surface representations.
*   **Limitations:** Requires accurate surface normals and roughness estimates. Performance depends on the quality of the underlying geometry reconstruction.
*   **When to use:** When modeling view-dependent reflectance (e.g., anisotropy, fresnel-like effects) in neural inverse rendering or neural point clouds.
*   **When not to use:** For purely view-independent diffuse materials or when computational resources are extremely constrained and simple analytic approximations suffice.

## Background
The blending scalar method originates from Photometric Inverse Rendering with Shading Cues Modeling (PIR) [S1]. It was introduced to regularize surface reflectance and model shading cues in joint optimization of geometry and lighting. The opacity activation formula ($\alpha_h = 1 - e^{-e^x}$) is adapted from prior work [43] cited in INPC [S2], which itself references spherical contraction techniques [4] [S2].

## Sources
[S1] PIR: Photometric Inverse Rendering with Shading Cues Modeling and Surface Reflectance Regularization, https://doi.org/10.1109/3dv66043.2025.00055, fetched 2026-10-26
[S2] INPC: Implicit Neural Point Clouds for Radiance Field Rendering, https://doi.org/10.1109/3dv66043.2025.00021, fetched 2026-10-26
