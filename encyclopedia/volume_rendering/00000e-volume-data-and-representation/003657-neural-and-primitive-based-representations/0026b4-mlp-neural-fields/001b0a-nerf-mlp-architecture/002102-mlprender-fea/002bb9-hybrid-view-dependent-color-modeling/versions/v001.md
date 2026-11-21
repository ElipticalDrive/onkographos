# Hybrid View-dependent Color Modeling

Also called: Feature Voxel Grid + Shallow MLP

## Overview
This method models view-dependent color emissions by combining a sparse feature voxel grid with a shallow Multi-Layer Perceptron (MLP) [S1]. It solves the trade-off between the slow convergence of pure neural fields and the limited expressivity of static voxel grids [S1]. Unlike its sibling [NeRF with Per-Image Embeddings](2484-nerf-with-per-image-embeddings.md) which uses global image vectors, or [Linear Feature Decoder with Learned Color Basis](3352-linear-feature-decoder-with-learned-color-basis.md) which uses linear projections, this hybrid approach uses a local spatial lookup followed by a non-linear shallow MLP to capture complex specularities and shading [S1].

## How it works
**Inputs:**
- Query position $\mathbf{x} \in \mathbb{R}^3$.
- View direction $\mathbf{v} \in \mathbb{R}^3$ (normalized).
- Feature grid resolution parameter $D$ (typically 12 or 24) [S1].

**Outputs:**
- RGB color vector $\mathbf{c} \in \mathbb{R}^3$.

**Data Structures:**
1.  **Feature Voxel Grid:** A 3D grid of dimensions $D \times D \times D$. Each cell stores a feature vector $\mathbf{f} \in \mathbb{R}^{C_f}$. The source specifies the grid stores "view-invariant diffused colors" or features, but does not explicitly define the channel count $C_f$ of the grid entries in the provided notes, only the MLP dimensions [S1]. [G] Standard implementations typically use $C_f=3$ or higher; however, based strictly on the provided text, the grid dimension is $D$ and the MLP takes features as input. We assume the grid stores features $\mathbf{f}$ that are concatenated with the view direction.
2.  **Shallow MLP:** A 2-layer MLP.
    -   Input dimension: $C_f + 3$ (features + view direction). Note: Positional encoding is applied to features and view direction, but NOT position, as per the parent [MLPRender_Fea](2414-mlprender-fea.md).
    -   Hidden layers: 2 layers, each with 128 channels.
    -   Activation: ReLU (implied by "shallow MLP" context and standard practice, though not explicitly named in S1 notes, S2 confirms ReLU usage in variants).
    -   Output layer: 1 layer, 3 channels (RGB).

**Numbered Steps:**
1.  **Grid Lookup:** Given query position $\mathbf{x}$, trilinearly interpolate the feature vector $\mathbf{f}$ from the voxel grid at resolution $D$.
2.  **View Direction Encoding:** Apply positional encoding to the view direction $\mathbf{v}$ [S1]. [G] The parent article specifies positional encoding is applied to view direction.
3.  **Feature Encoding:** Apply positional encoding to the interpolated feature vector $\mathbf{f}$ [S1].
4.  **Concatenation:** Concatenate the encoded features and encoded view direction: $\mathbf{z} = [\text{PE}(\mathbf{f}), \text{PE}(\mathbf{v})]$.
5.  **MLP Forward Pass:**
    -   $\mathbf{h}_1 = \text{ReLU}(\mathbf{W}_1 \mathbf{z} + \mathbf{b}_1)$
    -   $\mathbf{h}_2 = \text{ReLU}(\mathbf{W}_2 \mathbf{h}_1 + \mathbf{b}_2)$
    -   $\mathbf{c} = \mathbf{W}_3 \mathbf{h}_2 + \mathbf{b}_3$
    -   Where $\mathbf{W}_1, \mathbf{W}_2 \in \mathbb{R}^{128 \times \text{input\_dim}}$ and $\mathbf{W}_3 \in \mathbb{R}^{3 \times 128}$.
6.  **Output:** Return $\mathbf{c}$.

**Parameters:**
-   $D$: Feature grid dimension. Typical values: 12 (baseline), 24 (high-res) [S1].
-   MLP Layers: 2 [S1].
-   MLP Channels: 128 [S1].

**Edge Cases and Pitfalls:**
-   **Memory:** Using only the shallow MLP results in worse quality due to shallow depth compared to deeper NeRF MLPs [S1]. Using only the voxel grid converges faster but degrades quality [S1].
-   **Free Space Skipping:** To prevent Out-Of-Memory (OOM) errors during training, a free space skipping strategy is used. If the density at a coarse stage is below threshold $\tau^{(c)}$ or fine stage below $\tau^{(f)}$, the point is skipped [S1]. Typical thresholds: $\tau^{(c)} = 10^{-3}$, $\tau^{(f)} = 10^{-4}$ [S1]. Without skipping ($\tau=0$), the system runs out of memory at the fine stage [S1].

## Subtopics

### Signed Residual Activation
A variant used in texture atlas baking pipelines where a signed residual is added to a per-primitive base color inside the outer ReLU of the color equation [S2, unofficial].
-   **Formula:** $\text{ReLU}(\text{base} + b_{SV})$ [S2, unofficial].
-   **Positive-Only Variant:** $\text{ReLU}(\text{base} + b_{SV}) + \text{ReLU}(\rho)$ [S2, unofficial].
-   **Comparison:** The signed model allows both darkening and brightening, reproducing all non-negative corrections and handling darkening textures. The positive-only variant restricts correction to addition only, yielding a strictly smaller function class that fails where texture must darken the base [S2, unofficial].
-   **Performance:** Signed residual improves LPIPS on 10/11 scenes and PSNR on 10/11 by 0.31 dB (Mip-NeRF 360) and 0.19 dB (Deep Blending) [S2, unofficial]. Positive-only loses ~0.5 dB PSNR and 0.01 LPIPS on indoor scenes [S2, unofficial].
-   **Pitfall:** Positive-only optimization is unreliable; seed spread across runs is an order of magnitude larger than signed (e.g., stump spread 0.0773 vs 0.0042) [S2, unofficial].
-   **Usage:** Used in the texture atlas baking pipeline for view-dependent color representation [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
-   **Memory:** Scales with $D^3$ for the voxel grid. Higher $D$ increases memory usage significantly. The shallow MLP (2 layers, 128 channels) has negligible memory footprint compared to the grid [S1].
-   **Time:** Training time increases with $D$ and MLP channels. Baseline ($D=12$) takes 7.3 minutes; High-res ($D=24$) takes 8.8 minutes on the tested hardware/scene [S1]. Free space skipping reduces training time from OOM/long to ~7.3-7.8 minutes [S1].

**Measured Performance:**
-   **Baseline ($D=12$, 2 layers, 128 ch):** PSNR 35.54, Training 7.3 min [S1].
-   **High-res ($D=24$, 2 layers, 128 ch):** PSNR 35.90, Training 8.8 min [S1].
-   **Free Space Skipping Impact:** Skipping with $\tau^{(c)}=10^{-3}, \tau^{(f)}=10^{-4}$ yields PSNR 35.54 in 7.3 min, compared to 34.89 PSNR in 7.8 min with only $\tau^{(f)}$ applied, and OOM with no skipping [S1].

**Strengths and Limitations:**
-   **Strengths:** Super-fast convergence compared to pure neural fields [S1]. Better quality than voxel-grid-only methods [S1]. Efficient memory usage via free space skipping [S1].
-   **Limitations:** Shallower MLP yields worse quality than deeper NeRF MLPs if used alone [S1]. Larger $D$ and channels improve quality at higher computational cost [S1].

**When to Use:**
-   Use when fast convergence is required and view-dependent effects are moderate.
-   Use when memory is constrained but pure voxel grids are insufficient for quality [S1].

**When Not to Use:**
-   Do not use if high-fidelity complex view-dependent effects (e.g., strong specularities) are required and training time is not a constraint (use deeper MLPs or spherical harmonics) [S1].
-   Do not use without free space skipping if the scene has large empty volumes, as it may lead to OOM [S1].

## Background
Originated in the context of Direct Voxel Grid Optimization for radiance fields reconstruction [S1]. The method addresses the limitation of shallow MLPs in NeRF-like architectures by offloading spatial invariance to a voxel grid while retaining the MLP for view-dependent details [S1]. It is part of the MLPRender_Fea module family [S1].

## Sources
[S1] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
[S2] Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting, https://doi.org/10.48550/arXiv.2607.13808, fetched 2026-10-29 (unofficial)
