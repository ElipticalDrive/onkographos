# Relightable 3D Gaussian

Also called: Relightable 3D Gaussian Splatting

## Overview
Relightable 3D Gaussian extends the standard 3D Gaussian Splatting representation by assigning physically-based BRDF parameters (albedo, roughness, normal) to each Gaussian primitive, enabling dynamic relighting and shadow casting [S1, unofficial]. Unlike standard 3DGS which renders fixed colors, this method decouples material properties from lighting, allowing the scene to be re-rendered under arbitrary environment maps or light sources [S1, unofficial]. It distinguishes itself from sibling methods like [3D Gaussian Splatting with BRDF Embedding](1174-3d-gaussian-splatting-with-brdf-embedding.md) by explicitly baking precomputed visibility for shadows directly into the Gaussian attributes, rather than relying solely on learned embeddings or deferred shading passes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $\{G_i\}$ initialized from a standard 3DGS reconstruction, along with training images under varying lighting conditions (or a static light source for shadow baking).
*   **Output:** A scene representation where each Gaussian $i$ stores position $\mathbf{p}_i$, covariance $\Sigma_i$, opacity $\alpha_i$, spherical harmonics coefficients for albedo $\mathbf{c}_{albedo}$, roughness $\rho_i$, normal $\mathbf{n}_i$, and a precomputed visibility term $V_i$.

**Data Structures:**
Each Gaussian $i$ maintains the following fields:
1.  **Position:** $\mathbf{p}_i \in \mathbb{R}^3$
2.  **Covariance:** $\Sigma_i \in \mathbb{R}^{3 \times 3}$ (or quaternion/scale representation)
3.  **Opacity:** $\alpha_i \in [0, 1]$
4.  **Albedo:** $\mathbf{c}_{albedo}$ (Spherical Harmonics coefficients, typically degree 0-3)
5.  **Roughness:** $\rho_i \in [0, 1]$
6.  **Normal:** $\mathbf{n}_i \in \mathbb{R}^3$ (unit vector)
7.  **Visibility:** $V_i \in [0, 1]$ (precomputed shadow/occlusion factor)

**Method Steps:**
1.  **Initialization:** Initialize Gaussians using standard 3DGS SfM points. Assign initial albedo from the first training image's color. Set roughness to a default value (e.g., 0.5) and normals to the view direction or surface normal estimate [S1, unofficial].
2.  **Visibility Baking:** To handle shadows, precompute visibility $V_i$ for each Gaussian. This involves casting rays from the light source to each Gaussian center. If occluded, $V_i$ is reduced. This step is performed once or updated periodically during optimization [S1, unofficial].
3.  **Rendering:** For a given ray $\mathbf{r}$ and light environment $\mathcal{L}$:
    *   Project Gaussians to 2D and sort by depth.
    *   For each Gaussian $i$ along the ray, compute the outgoing radiance $L_o$ using the BRDF $f_r$:
        $$
        L_o = V_i \cdot \int_{\Omega} f_r(\omega_i, \omega_o) L_i(\omega_i) (\omega_i \cdot \mathbf{n}_i) d\omega_i
        $$
        where $L_i$ is the incident light, $\omega_i$ is the incoming light direction, and $\omega_o$ is the view direction.
    *   Composite colors using alpha blending:
        $$
        \hat{C} = \sum_{i \in \mathcal{N}} T_i \alpha_i \hat{C}_i
        $$
        where $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ is the accumulated transmittance, and $\hat{C}_i$ is the color contribution from Gaussian $i$ under the current lighting.
4.  **Optimization:** Minimize the photometric loss $\mathcal{L}$ between rendered and ground truth images. Gradients flow through the BRDF parameters and visibility. The opacity $\alpha_i$ and covariance $\Sigma_i$ are also optimized as in standard 3DGS [S1, unofficial].

**Parameters:**
*   **Roughness Range:** $[0, 1]$
*   **Visibility Update Frequency:** Not given in the sources. [G] Typically updated every $K$ steps or per epoch to balance performance and accuracy.
*   **SH Degree for Albedo:** Typically 3 (degree 0-3) to capture directional albedo variations.

**Edge Cases:**
*   **Degenerate Normals:** If normal estimation is poor, reflections may appear incorrect. [G] Use normal regularization or fallback to view-dependent colors if confidence is low.
*   **Shadow Artifacts:** Precomputed visibility may cause aliasing if Gaussians are small relative to shadow boundaries. [G] Consider softening visibility or using higher-resolution shadow maps.

## Subtopics

### R3DG
R3DG is a variant that combines BRDF parameters with ambient occlusion baked per Gaussian [S1, unofficial]. It extends the basic relightable model by incorporating occlusion information directly into the Gaussian attributes, improving the realism of indirect lighting effects without requiring complex global illumination calculations. It is chosen when ambient occlusion is critical for material appearance but full path tracing is too expensive.

## Operating characteristics
*   **Time Complexity:** Rendering is similar to standard 3DGS, $O(N)$ per pixel where $N$ is the number of Gaussians. However, the BRDF evaluation adds constant overhead per Gaussian. Visibility baking is $O(N \cdot L)$ where $L$ is the number of light samples, but is done offline or periodically.
*   **Memory Complexity:** Higher than standard 3DGS due to additional attributes (roughness, normal, visibility). Each Gaussian requires extra storage for BRDF parameters and visibility.
*   **Performance:** Enables real-time relighting if the environment map is pre-filtered or low-resolution. High-frequency reflections may require higher bandwidth SH or pre-filtered environment maps, as SH-only is insufficient for sharp reflections [S1, unofficial].
*   **Strengths:** Physically plausible materials, dynamic lighting, shadow support.
*   **Limitations:** Requires training under varying lighting or careful visibility baking. May struggle with complex indirect lighting without additional techniques.

## Background
Relightable 3D Gaussian was introduced by Gao et al. in ECCV'24 [S1, unofficial]. It builds upon the 3D Gaussian Splatting framework to address the limitation of fixed-color rendering by introducing material properties. It is used in applications requiring dynamic lighting, such as virtual production, AR/VR, and interactive 3D editing.

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
