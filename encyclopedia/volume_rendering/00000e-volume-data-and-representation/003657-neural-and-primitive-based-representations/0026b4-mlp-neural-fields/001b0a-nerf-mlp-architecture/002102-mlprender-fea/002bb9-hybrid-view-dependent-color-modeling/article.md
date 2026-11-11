# Hybrid View-dependent Color Modeling

Also called: Feature Voxel Grid + Shallow MLP

## Overview
Hybrid View-dependent Color Modeling predicts RGB color by combining a sparse feature voxel grid with a shallow Multi-Layer Perceptron (MLP) [S1]. It solves the trade-off between the slow convergence of pure neural fields and the limited quality of explicit voxel grids by storing view-independent diffuse color and view-dependent features in the grid, while using the MLP to decode view-dependent residuals [S1, S3]. It sets itself apart from [MLPRender_Fea](2414-mlprender-fea.md) by explicitly separating the view-independent component (stored in the grid) from the view-dependent component (decoded by the MLP), whereas [MLPRender_Fea](2414-mlprender-fea.md) applies positional encoding to features and view direction but not position [2414-mlprender-fea.md].

## How it works
**Inputs:**
*   3D position $\mathbf{x}$.
*   View direction $\mathbf{v}$.
*   Feature grid $G$ with dimension $D$ (typically 12 or 24) [S1].
*   Shallow MLP with 2 layers and 128 channels per layer [S1].

**Outputs:**
*   RGB color $\mathbf{C}$.

**Data Structures:**
*   **Feature Voxel Grid:** A sparse 3D grid storing feature vectors of dimension $D$ and view-independent diffuse color [S1, S3].
*   **Shallow MLP:** A 2-layer MLP with input dimension $D + 3$ (features + view direction), hidden layer size 128, and output dimension 3 (RGB residual) [S1].

**Steps:**
1.  Query the feature voxel grid at position $\mathbf{x}$ to retrieve the feature vector $\mathbf{f}$ [S1, S3].
2.  Retrieve the view-independent diffuse color $\mathbf{C}_{\text{diff}}$ from the grid at $\mathbf{x}$ [S3].
3.  Concatenate $\mathbf{f}$ and the view direction $\mathbf{v}$ to form the input vector $\mathbf{z} = [\mathbf{f}; \mathbf{v}]$ [S3].
4.  Pass $\mathbf{z}$ through the shallow MLP to compute the view-dependent residual $\mathbf{r}$ [S1, S3].
5.  Compute the final color $\mathbf{C} = \mathbf{C}_{\text{diff}} + \mathbf{r}$ [S3].

**Parameters:**
*   Feature grid dimension $D$: 12 or 24 [S1].
*   MLP layers: 2 [S1].
*   MLP channels: 128 [S1].

**Edge Cases and Pitfalls:**
*   **Free Space Skipping:** To avoid out-of-memory errors and improve convergence, a free space skipping strategy queries points only if they pass density thresholds in coarse and fine stages [S1].
*   **MLP Depth:** Using only the shallow MLP yields worse quality because it is shallower than the full NeRF MLP (2 layers vs 8 layers) [S1].
*   **Voxel Grid Only:** Using only the voxel grid converges faster but degrades rendering quality [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def concatenate(a, b):
    return a + b

def hybrid_view_dependent_color(x, v, feature_grid, mlp):
    f = feature_grid.query(x)
    C_diff = feature_grid.query_color(x)
    z = concatenate(f, v)
    r = mlp.forward(z)
    C = C_diff + r
    return C
```

## Subtopics

### Signed Residual Activation
Applies a signed residual to a per-primitive base color, entering inside the outer ReLU of the color equation to allow both darkening and brightening [S2, unofficial]. The formula is $\text{ReLU}(\text{base} + b_{SV})$ [S2, unofficial]. A positive-only restriction $\text{ReLU}(\text{base} + b_{SV}) + \text{ReLU}(\rho)$ yields a strictly smaller function class that fails where texture must darken the base [S2, unofficial]. Signed residual improves LPIPS on 10/11 scenes and PSNR on 10/11 by 0.31 dB (Mip-NeRF 360) and 0.19 dB (Deep Blending) [S2, unofficial]. Positive-only makes optimization unreliable; seed spread across runs is an order of magnitude larger (e.g., stump spread 0.0773 vs 0.0042) [S2, unofficial]. Used in the texture atlas baking pipeline [S2, unofficial].

### Baking Neural Fields
Precomputes view-independent diffuse RGB color and view-dependent 4D feature vectors, storing them in a sparse voxel grid and texture atlas [S3]. Steps: accumulate diffuse colors and feature vectors along a ray; concatenate with view direction; pass to shallow MLP for view-dependent residual; add residual to diffuse color [S3]. Averages less than 90 MB per scene and renders in real-time (>30 fps on laptop GPU for 800x800) [S3]. Image compression leads to relatively low-quality metrics [S3].

### View-Independent SH Color (Degree 0)
Omits view-dependent spherical harmonics (SH) coefficients by setting SH degree to 0, using only view-independent color [S4, unofficial]. On Mip-NeRF360 outdoor scenes (RTX 4090), reduces training time from 5m31s to 2m24s, VRAM from 6.0GiB to 3.2GiB, and PSNR drops from 24.72 to 24.38 [S4, unofficial]. Speeds up training and reduces VRAM but slightly degrades quality [S4, unofficial].

### View-Disparity Color Prediction
Uses view disparity $\delta v_i$ as a parameter, representing the difference between the relative viewing direction of the target view and input view $i$ [S5]. A smaller difference indicates higher similarity between the target view color and the color corresponding to input view $i$ [S5].

### Sparse Neural Voxel Grid (SNeRG)
A model storing a precomputed NeRF on a sparse voxel grid, sometimes referred to as 'baking' [S6]. During evaluation, an MLP produces specular color [S6]. The specular color is combined with alpha compositing of the specular color along the ray to yield the final pixel color [S6]. The layout uses a sparse voxel grid storing precomputed diffuse color, density, and feature vectors [S6]. It is approximately 3000 times faster than the original NeRF implementation [S6].

## Operating characteristics
*   **Time Complexity:** Faster than full NeRF MLP due to shallow MLP (2 layers) [S1].
*   **Memory Complexity:** Compact, averaging less than 90 MB per scene when baked [S3].
*   **Performance:** Baseline ($D=12$, 2 layers, 128 ch): PSNR 35.54, 7.3 min training. High-res ($D=24$, 2 layers, 128 ch): PSNR 35.90, 8.8 min training [S1].
*   **Strengths:** Fast convergence, compact storage, real-time rendering capability when baked [S1, S3].
*   **Limitations:** Lower quality than full NeRF MLP if MLP is too shallow or voxel grid is used alone [S1]. Image compression for storage reduces quality [S3].

## Background
Originated from the need to accelerate NeRF rendering by precomputing view-independent components and deferring view-dependent computations [S3]. Hedman et al. [HSM* 21] reformulated NeRF to output diffuse color and view-dependent features [S3]. FastNeRF [GKJ* 21] cached view-independent radiance in sparse voxel grids [S3]. Yu et al. [YLT* 21] introduced NeRF-SH for view-dependent appearance [S3]. The hybrid approach combines these ideas with a shallow MLP for efficient view-dependent decoding [S1].

## Sources
[S1] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
[S2] Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting, https://doi.org/10.48550/arXiv.2607.13808, fetched 2026-10-29 (unofficial)
[S3] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S4] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
[S5] Rugularizing generalizable neural radiance field with limited-view images, https://link.springer.com/article/10.1007/s40747-024-01696-6, fetched 2026-10-29
[S6] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
