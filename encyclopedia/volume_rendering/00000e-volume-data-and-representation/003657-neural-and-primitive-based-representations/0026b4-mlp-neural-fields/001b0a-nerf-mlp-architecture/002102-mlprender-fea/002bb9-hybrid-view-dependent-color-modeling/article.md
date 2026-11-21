# Hybrid View-dependent Color Modeling

Also called: Feature Voxel Grid + Shallow MLP

## Overview
Hybrid View-dependent Color Modeling predicts RGB color by combining a sparse feature voxel grid with a shallow Multi-Layer Perceptron (MLP), addressing the trade-off between the slow convergence of pure neural fields and the limited quality of grid-only representations [S1]. Unlike [NeRF with Per-Image Embeddings](2484-nerf-with-per-image-embeddings.md) which uses global image vectors, or [Volume Radiance Network](2576-volume-radiance-network.md) which relies on dense MLPs, this method decouples view-independent diffuse color storage from view-dependent shading computation, enabling faster training and real-time rendering capabilities [S1, S3].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D position $\mathbf{x}$, view direction $\mathbf{v}$, and a precomputed feature vector $\mathbf{f}$ retrieved from a voxel grid at $\mathbf{x}$.
*   **Output:** RGB color $\mathbf{c}$.

**Data Structures:**
*   **Feature Voxel Grid:** A 3D grid of dimensions $D \times D \times D$. Each voxel stores a feature vector. Typical dimensions $D$ are 12 or 24 [S1]. The grid stores view-independent diffuse colors and feature vectors [S3].
*   **Shallow MLP:** A feedforward network with 2 hidden layers, each containing 128 channels [S1].

**Numbered Steps:**
1.  **Feature Retrieval:** Query the feature voxel grid at the 3D position $\mathbf{x}$ to obtain the feature vector $\mathbf{f}$ and the base diffuse color $\mathbf{c}_{diffuse}$ [S3].
2.  **Concatenation:** Concatenate the retrieved feature vector $\mathbf{f}$ with the view direction $\mathbf{v}$ to form the input tensor $\mathbf{z} = [\mathbf{f}, \mathbf{v}]$ [S3].
3.  **MLP Forward Pass:** Pass $\mathbf{z}$ through the shallow MLP (2 layers, 128 channels) to predict a view-dependent residual color $\mathbf{c}_{residual}$ [S1, S3].
4.  **Color Composition:** Add the residual to the base diffuse color: $\mathbf{c} = \mathbf{c}_{diffuse} + \mathbf{c}_{residual}$ [S3].

**Formulas:**
The final color is computed as:
$$ \mathbf{c} = \mathbf{c}_{diffuse} + \text{MLP}([\mathbf{f}, \mathbf{v}]) $$
Where $\mathbf{c}_{diffuse}$ is the view-independent component stored in the grid, and $\mathbf{f}$ is the view-dependent feature vector also stored in the grid [S3].

**Parameters:**
*   **MLP Layers:** 2 [S1].
*   **MLP Channels:** 128 [S1].
*   **Grid Dimension ($D$):** 12 or 24 [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Using only the shallow MLP results in worse quality due to its shallow depth compared to standard NeRF MLPs (8 layers, 256 channels) [S1].
*   **Memory:** Without free-space skipping, fine-stage voxel grids may run out of memory (OOM) [S1].
*   **Degenerate Input:** If the grid dimension $D$ is too small, rendering quality degrades significantly [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hybrid_view_dependent_color(position, view_direction, voxel_grid, mlp):
    """
    Predicts RGB color using hybrid view-dependent modeling.
    
    Args:
        position (float3): 3D position (x, y, z).
        view_direction (float3): Normalized view direction (vx, vy, vz).
        voxel_grid (VoxelGrid): Sparse 3D grid storing features and diffuse color.
        mlp (MLP): Shallow MLP with 2 layers, 128 channels.
        
    Returns:
        float3: Final RGB color.
    """
    # Step 1: Retrieve feature vector and diffuse color from voxel grid
    # Note: Interpolation method (e.g., trilinear) depends on implementation.
    feature_vector = voxel_grid.query(position)
    diffuse_color = voxel_grid.get_diffuse_color(position)
    
    # Step 2: Concatenate feature vector with view direction
    # Input to MLP is [feature_vector, view_direction]
    input_tensor = concatenate(feature_vector, view_direction)
    
    # Step 3: Pass through shallow MLP to get residual
    residual_color = mlp.forward(input_tensor)
    
    # Step 4: Add residual to diffuse color
    final_color = diffuse_color + residual_color
    
    return final_color
```

## Subtopics

### [View-Independent SH Color (Degree 0)](9600-view-independent-sh-color-degree-0.md)
This method simplifies neural field color representation by restricting spherical harmonics (SH) to degree 0, thereby modeling color as purely view-independent. It solves the problem of high computational overhead and memory usage associated with view-dependent shading in neural radiance fields. Unlike its sibling [DCGO](15557-dcgo-deep-convolutional-grid-optimization.md), which optimizes grid structures for view-dependent features, this approach explicitly discards directional information to accelerate training and reduce VRAM consumption at the cost of slight quality degradation.

### [DCGO (Deep Convolutional Grid Optimization)](15557-dcgo-deep-convolutional-grid-optimization.md)
DCGO is a hybrid volume representation that extends the standard Direct Volume Grid Optimization (DVGO) by introducing space contraction and optimized ray sampling to handle unbounded or inward-facing scenes. It replaces the linear depth sampling of its sibling [DVGO](5082-dvgo.md) with a contracted coordinate system and a two-stage sampling strategy (inner/outer), allowing for efficient rendering of scenes with infinite background or specific camera constraints.

### Signed Residual Activation
Applies a signed residual to a per-primitive base color, entering inside the outer ReLU of the color equation to allow both darkening and brightening [S2, unofficial]. The formula is $\text{ReLU}(\text{base} + b_{SV})$. A positive-only restriction uses $\text{ReLU}(\text{base} + b_{SV}) + \text{ReLU}(\rho)$, which yields a strictly smaller function class that fails wherever the texture must darken the base [S2, unofficial]. The signed residual improves LPIPS on 10/11 scenes and PSNR on 10/11 by 0.31 dB (Mip-NeRF 360) and 0.19 dB (Deep Blending) [S2, unofficial]. Positive-only makes optimization unreliable; seed spread across runs is an order of magnitude larger than signed (e.g., stump spread 0.0773 vs 0.0042) [S2, unofficial]. Used in the texture atlas baking pipeline for view-dependent color representation [S2, unofficial].

### Baking Neural Fields
A method that precomputes view-independent diffuse RGB color and view-dependent 4D feature vectors, storing them in a sparse voxel grid and texture atlas [S3]. Steps include: 1) Accumulate diffuse colors and feature vectors along a ray; 2) Concatenate accumulated values with the view direction; 3) Pass concatenated values to a shallow MLP to produce a view-dependent residual; 4) Add the view-dependent residual to the diffuse color [S3]. Image compression algorithms used to reduce storage size lead to relatively low-quality metrics [S3]. Averages less than 90 MB per scene and renders in real-time (higher than 30 fps on a laptop GPU for 800x800 images) [S3].

### View-Disparity Color Prediction
Uses $\delta v_i$, the view disparity, representing the difference between the relative viewing direction of the target view and input view $i$. A smaller difference indicates higher similarity [S4].

### Sparse Neural Voxel Grid (SNeRG)
A model storing a precomputed NeRF on a sparse voxel grid, sometimes referred to as 'baking' [S5]. During evaluation, an MLP produces specular color, which is combined with alpha compositing of the specular color along the ray to yield final pixel color [S5]. The layout stores precomputed diffuse color, density, and feature vectors in a sparse voxel grid [S5]. It is approximately 3000 times faster than the original NeRF implementation [S5].

## Operating characteristics
*   **Time Complexity:** Faster than pure MLP-based NeRF due to the shallow MLP and grid lookups. Training converges faster than grid-only methods [S1].
*   **Memory Complexity:** Depends on grid dimension $D$. Larger $D$ increases memory usage but improves quality [S1]. Free-space skipping strategies can reduce memory consumption [S1].
*   **Measured Performance:** Baseline ($D=12$, 2 layers, 128 ch) achieves PSNR 35.54 in 7.3 minutes. High-res ($D=24$, 2 layers, 128 ch) achieves PSNR 35.90 in 8.8 minutes [S1].
*   **Strengths:** Balances quality and speed; enables real-time rendering when baked [S3].
*   **Limitations:** Quality degrades if grid resolution is too low; pure MLP is insufficient due to shallow depth [S1].
*   **When to use:** When training time and inference speed are critical, but view-dependent effects are still desired [S1, S3].
*   **When not to use:** When maximum possible quality is required regardless of speed, or when scene geometry is extremely complex requiring deep MLPs [S1].

## Background
Originated from the need to accelerate NeRF training and rendering. The hybrid approach combines the benefits of voxel grids (fast lookup, view-independent storage) and shallow MLPs (view-dependent modeling) [S1, S3]. It was developed as an alternative to spherical harmonics and dense MLPs, offering a middle ground in terms of computational cost and visual fidelity [S1]. The method is closely related to baking techniques where neural fields are precomputed into spatial structures [S3, S5].

## Sources
[S1] Direct Voxel Grid Optimization: Super-fast Convergence for Radiance Fields Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00538, fetched 2026-10-26
[S2] Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting, https://doi.org/10.48550/arXiv.2607.13808, fetched 2026-10-29 (unofficial)
[S3] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S4] Rugularizing generalizable neural radiance field with limited-view images, https://link.springer.com/article/10.1007/s40747-024-01696-6, fetched 2026-10-29
[S5] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
