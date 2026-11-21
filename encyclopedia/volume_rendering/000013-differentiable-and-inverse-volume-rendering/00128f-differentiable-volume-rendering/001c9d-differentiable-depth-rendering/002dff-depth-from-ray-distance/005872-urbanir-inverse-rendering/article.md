# UrbanIR Inverse Rendering

Also called: Inverse Rendering with Visibility Guidance

## Overview
UrbanIR Inverse Rendering is a differentiable volume rendering method that jointly optimizes a neural scene representation and lighting parameters from monocular video to recover physically plausible albedo, normals, semantics, and shadows. It solves the ill-posed nature of inverse rendering in urban environments by computing visibility via volume rendering transmittance rather than explicit geometry, ensuring consistency between the learned density field and shadow casting. Unlike sibling methods that focus on depth estimation or expected terms, this method explicitly models the interaction between sunlight and scene geometry to enable realistic relighting and shadow removal.

## How it works
The method trains a neural field $F(\cdot)$ representing density $\sigma$ and color components (albedo $\mathbf{a}$, normal $\mathbf{n}$, semantic $\mathbf{s}$) and a lighting model $\mathcal{L}$ jointly. The core innovation is the visibility estimation for directional sunlight using the volume rendering framework itself.

**Inputs and Outputs:**
*   **Inputs:** Camera rays $\mathcal{R}$, ground-truth RGB images $C_{gt}$, shadow masks $M$ (from external detector [MTMT]), semantic priors, normal priors.
*   **Outputs:** Rendered color $C(r)$, albedo $A$, normal $N$, semantic $S$, visibility $V$, lighting parameters (ambient $L_{amb}$, sky color $C_{sky}$).

**Data Structures:**
The neural field $F$ maps 3D coordinates and viewing directions to density and color attributes.
*   Density: $\sigma \in \mathbb{R}$
*   Albedo: $\mathbf{a} \in \mathbb{R}^3$
*   Normal: $\mathbf{n} \in \mathbb{R}^3$
*   Semantic: $\mathbf{s} \in \mathbb{R}^k$ (where $k$ is number of classes)

**Numbered Steps:**

1.  **Volume Rendering of Intrinsics:** For each camera ray $r$, sample points $x_i$ along the ray. Compute density $\sigma_i$ and attributes $\mathbf{a}_i, \mathbf{n}_i, \mathbf{s}_i$ using $F$. Compute alpha-composition weights:
    $$ w_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) (1 - \exp(-\sigma_i \delta_i)) $$
    where $\delta_i = t_i - t_{i-1}$. Render the final color, normal, and semantic maps:
    $$ C(r) = \sum_{i=1}^{N} w_i \mathbf{a}_i, \quad N(r) = \sum_{i=1}^{N} w_i \mathbf{n}_i, \quad S(r) = \sum_{i=1}^{N} w_i \mathbf{s}_i $$
    [S1]

2.  **Depth Estimation for Shading Point:** Compute the expected depth $\hat{t}$ for the ray to determine the 3D point $x(r)$ being shaded:
    $$ \hat{t} = \sum_{i=1}^{N} w_i t_i $$
    Substitute $\hat{t}$ into the ray equation to get the 3D position $x(r)$. [S1]

3.  **Visibility Estimation via Volume Rendering:** To estimate visibility $V(x, \text{sun})$ of point $x$ to the sun, cast a ray from $x$ in the direction of the sun $\mathbf{l}_{\text{sun}}$. Compute transmittance along this segment using the same neural field:
    $$ V(x, \text{sun}) = \exp\left(-\sum_i \sigma_i(x_i) \delta_i\right) \quad \text{where } x_i = x + t_i \mathbf{l}_{\text{sun}} $$
    [S1]

4.  **Guidance Visibility Head:** To avoid instability from direct 2D shadow mask supervision (which is inconsistent across frames), train an intermediate MLP head $\hat{V}(r)$ to reproduce the 2D shadow masks $M(r)$. This provides a consistent guidance signal. [S1]

5.  **Loss Computation:** Compute the total loss $\mathcal{L}$:
    $$ \mathcal{L} = \mathcal{L}_{\text{render}} + \lambda_1 \mathcal{L}_{\text{visibility}} + \lambda_2 \mathcal{L}_{\text{normal}} + \lambda_3 \mathcal{L}_{\text{semantics}} + \lambda_4 \mathcal{L}_{\text{reg}} $$
    [S1]

**Formulas:**
*   **Rendering Loss:**
    $$ \mathcal{L}_{\text{render}} = \sum_{r} \| C_{gt}(r) - C(r) \|_2^2 $$
    [S1]
*   **Visibility Loss:** The loss encourages $V(x, \text{sun})$ to be low where the guidance head $\hat{V}(r)$ predicts shadow, and high otherwise. Specifically, it constrains geometry along the ray from the pixel to the sun based on shadow masks. [S1]
*   **Normal Loss:** $\mathcal{L}_{\text{normal}}$ supervises the rendered normal $N(r)$ against priors from Omnidata. [S1]
*   **Semantic Loss:** $\mathcal{L}_{\text{semantics}}$ supervises the rendered semantic $S(r)$ against priors. [S1]
*   **Regularization Loss:**
    $$ \mathcal{L}_{\text{reg}} = \mathcal{L}_{\text{albedo}} + \| L_{\text{amb}} \|_2^2 $$
    where $\mathcal{L}_{\text{albedo}}$ regularizes ground albedo to be homogeneous:
    $$ \mathcal{L}_{\text{albedo}} = \sum_{r \in \text{ground}} \| A(r) - \bar{A}_g \|_2^2 $$
    and $\bar{A}_g$ is the average ground albedo. $\| L_{\text{amb}} \|_2^2$ prevents unnatural color shifts from large ambient light intensity. [S1]

**Parameters:**
*   $\lambda_1, \lambda_2, \lambda_3, \lambda_4$: Weights for visibility, normal, semantics, and regularization losses. Typical values are not specified in the sources. [S1]

**Pitfalls:**
*   **Instability from 2D Masks:** Direct supervision of visibility with 2D shadow masks is unstable due to inconsistency across frames. Solved by using the intermediate MLP head $\hat{V}$ for guidance. [S1]
*   **Ill-posed Optimization:** Large ambient light intensity causes unnatural color shifts in recovered albedo. Solved by regularizing $\| L_{\text{amb}} \|_2^2$. [S1]
*   **Inconsistency with Explicit Geometry:** Explicit geometry shadow casting may be inconsistent with neural fields. This method avoids this by computing visibility via volume rendering transmittance. [S1]

## Subtopics

### UrbanIR Training Pipeline
The training pipeline retrieves scene intrinsics (albedo, normal, semantic) via volume rendering from camera rays, guided by semantic and normal priors. Transmittance along tracing rays is supervised with shadow masks. Inputs include the input image $I$, Instruct-Pix2Pix output $I_{p2p}$, shadow mask $M$ from MTMT, RGB ground truth $C_{gt}$, albedo $A$, and sky color $C_{sky}$. The pipeline uses losses $\mathcal{L}_{render}$ and $\mathcal{L}_{visibility}$ for volume rendering (Eq. 4 and 5), and $\mathcal{L}_{seman/c}$, $\mathcal{L}_{normal}$, $\mathcal{L}_{reg}$ for intrinsic decomposition. Priors come from Normal [Omnidata] and Semantic models. [S1]

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Evaluated on KITTI-360 and Waymo Open Dataset. The method successfully removes dark shadows with sharp boundaries and enables realistic nighttime rendering by introducing new light sources. [S1]
*   **Scaling:** Designed for large-scale urban scenes from single video. [S1]
*   **Strengths:** Consistency between learned density field and shadow casting; effective shadow removal in albedo; handles relighting. [S1]
*   **Limitations:** Ill-posed nature requires strong regularization; dependency on external shadow masks and priors. [S1]
*   **When to use:** When recovering physically plausible inverse rendering results (albedo, normals, shadows) from monocular urban video is required. [S1]
*   **When not:** When real-time performance is critical or when explicit geometry is preferred over neural fields. [S1]

## Background
UrbanIR was introduced in the paper "UrbanIR: Large-Scale Urban Scene Inverse Rendering from a Single Video" published in 3DV 2025. It addresses the challenges of inverse rendering in urban driving sequences, where shadows and lighting variations complicate the recovery of intrinsic scene properties. The method builds on differentiable volume rendering and integrates semantic and normal priors to guide the optimization. [S1]

## Sources
[S1] UrbanIR: Large-Scale Urban Scene Inverse Rendering from a Single Video, https://doi.org/10.1109/3dv66043.2025.00052, fetched 2026-10-30
