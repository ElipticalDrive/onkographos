# Grid-NeRF

Also called: Grid-guided NeRF, Grid-guided Neural Radiance Fields (GridNeRF), Two-branch model

## Overview
Grid-NeRF is a two-branch architecture designed to render large-scale urban scenes by integrating grid-based and Neural Radiance Field (NeRF) approaches under a joint learning scheme [S1]. It solves the problem of limited model capacity in purely MLP-based NeRFs (which cause blurring) and noisy artifacts in pure feature grid methods when applied to massive scenes [S1]. It sets itself apart from its siblings by using a coarse feature grid to guide the NeRF branch's sampling and feature injection, rather than relying solely on hierarchical hash encoding or surface-relative constraints [S1].

## How it works
Grid-NeRF consists of two parallel branches: a **Grid Branch** and a **NeRF Branch**, which are optimized jointly [S1]. The method relies on a factorized 3D feature grid to reduce memory complexity from $O(N^3)$ to $O(N^2)$ [S1].

### Inputs and Outputs
*   **Inputs:** 3D spatial coordinates $\mathbf{x} = (x, y, z)$, viewing direction $\mathbf{d}$, and camera pose information (for position encoding).
*   **Outputs:** Volume density $\sigma$ and color $\mathbf{c}$ for each branch, combined to produce the final rendered pixel color.

### Data Structures
**Factorized Feature Grid:**
Unlike standard 3D voxel grids, Grid-NeRF factorizes the 3D feature grid into:
1.  **2D Ground Feature Planes:** A set of feature planes spanning the $xy$-plane (ground level) [S1]. Let these be $\mathbf{G}_{xy}$.
2.  **Vertical Shared Vector:** A feature vector shared along the $z$-axis [S1]. Let this be $\mathbf{V}_z$.
This factorization exploits the observation that urban semantics are primarily distributed on the ground [S1].

**Neural Networks:**
1.  **Grid Encoder:** Retrieves features from $\mathbf{G}_{xy}$ and $\mathbf{V}_z$ based on $\mathbf{x}$ [S1].
2.  **Lightweight MLP:** A small Multi-Layer Perceptron that maps the concatenated features and positional encodings to density $\sigma$ and color $\mathbf{c}$ [S1].

### Steps

**1. Pre-training Stage (Grid Branch Initialization):**
The target scene is modeled using the factorized feature grid to coarsely capture scene geometry and appearance [S1]. This provides a compact representation of the large scene [S1].

**2. Guidance Step 1 (Sampling Guidance):**
The coarse feature grid guides the NeRF branch's point sampling. Instead of uniform sampling along the ray, sampling concentrates around the scene surface, as inferred from the grid's density/geometry cues [S1]. This drastically compresses the sampling space [S1].

**3. Guidance Step 2 (Feature Injection):**
At each sampled position $\mathbf{x}_i$ along the ray:
*   Retrieve features from the factorized grid: $\mathbf{f}_{grid} = \text{GridRetrieve}(\mathbf{x}_i, \mathbf{G}_{xy}, \mathbf{V}_z)$ [S1].
*   Compute standard NeRF positional encoding: $\mathbf{PE}(\mathbf{x}_i)$ [S1].
*   Concatenate: $\mathbf{F}_{input} = [\mathbf{PE}(\mathbf{x}_i); \mathbf{f}_{grid}]$ [S1].
This supplies NeRF with extra features about scene geometry and appearance at sampled positions [S1].

**4. Joint Learning Stage:**
Both branches are optimized simultaneously using gradients from the final rendered image loss [S1].
*   The **NeRF Branch** uses the lightweight MLP to predict $\sigma$ and $\mathbf{c}$ from $\mathbf{F}_{input}$ and $\mathbf{d}$ [S1].
*   The **Grid Branch** is updated via backpropagation from the NeRF branch, regularizing the grid to produce more accurate and natural rendering results [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def grid_nerf_forward(x, d, grid_xy, vec_z, mlp_weights, camera_pos):
    """
    Forward pass for Grid-NeRF.
    x: 3D coordinate (x, y, z)
    d: Viewing direction
    grid_xy: 2D ground feature planes
    vec_z: Vertical shared feature vector
    mlp_weights: Weights for the lightweight MLP
    camera_pos: Camera position for positional encoding
    """
    
    # 1. Retrieve Grid Features
    # Factorized retrieval: combine xy-plane features and z-axis vector
    # Note: Specific interpolation for factorized grid is implementation-dependent [G]
    f_grid = retrieve_factorized_features(x, grid_xy, vec_z)
    
    # 2. Compute Positional Encoding
    # Standard NeRF positional encoding using camera position and x
    pe = positional_encoding(camera_pos, x)
    
    # 3. Concatenate Features
    # Input to MLP: [Positional Encoding; Grid Features]
    f_input = concatenate([pe, f_grid])
    
    # 4. MLP Prediction
    # Lightweight MLP outputs density and color
    sigma, color = mlp_forward(f_input, d, mlp_weights)
    
    return sigma, color

def joint_training_step(ray, grid_xy, vec_z, mlp_weights, loss_fn):
    """
    Single training step optimizing both grid and MLP.
    """
    # Sample points along ray, guided by grid geometry [S1]
    points = sample_points_guided_by_grid(ray, grid_xy)
    
    sigmas = []
    colors = []
    
    for p in points:
        sigma, color = grid_nerf_forward(p, ray.direction, grid_xy, vec_z, mlp_weights, ray.origin)
        sigmas.append(sigma)
        colors.append(color)
        
    # Volume Rendering
    rendered_color = volume_render(sigmas, colors, ray)
    
    # Compute Loss
    loss = loss_fn(rendered_color, ray.gt_color)
    
    # Backpropagate to update both grid and MLP
    loss.backward()
    
    return loss
```

### Parameters
*   **Grid Resolution:** Not given in the sources.
*   **MLP Depth/Width:** "Lightweight" / "Small" [S1][S2, unofficial]. Exact dimensions not given.
*   **Memory Complexity:** Reduced from $O(N^3)$ to $O(N^2)$ due to factorization [S1].

### Edge Cases and Pitfalls
*   **Large Scale Scenes:** Pure NeRF methods underfit and blur; pure grid methods show noisy artifacts [S1]. Grid-NeRF mitigates both.
*   **Factorization Assumption:** The method assumes urban semantics are primarily on the ground ($xy$-plane). This may not hold for non-urban or vertically complex scenes [S1].

## Subtopics

### HashNeRF
Uses hierarchical hash tables for multi-resolution encoding without explicit grid factorization. Choose when memory efficiency is critical and scene is not strictly urban/ground-dominant [8678-hashnerf.md].

### Surface-Relative Hash Encoding
Encodes features relative to a surface mesh rather than absolute coordinates. Choose when explicit geometry is available and surface detail is paramount [18178-surface-relative-hash-encoding.md].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** $O(N^2)$ due to factorization of 3D grid into 2D planes and shared vector [S1].
*   **Performance:** On a 2.7km² urban scene with 5k drone images, Grid-NeRF achieves PSNR 24.893 (Grid & NeRF branches) and 24.970, compared to NeRF-based PSNR 20.508 and Feature Grid-based PSNR 21.511 [S1].
*   **Strengths:** Photorealistic novel view synthesis for large-scale scenes; handles high-frequency details better than pure grids; avoids blurring of pure NeRFs [S1].
*   **Limitations:** Requires joint optimization of two branches; factorization may limit applicability to non-urban scenes [S1].
*   **When to use:** Large urban scene reconstruction from aerial/drone imagery [S1].
*   **When not to use:** Small-scale scenes where overhead is unjustified; non-urban scenes where ground-dominant assumption fails [S1].

## Background
Grid-NeRF was proposed by researchers from CUHK, MPI Informatics, Zhejiang University, and Adobe Research [S1]. It was published in CVPR 2023 [S1]. The work addresses the limitations of existing NeRF-based and feature grid-based methods for large-scale urban scene rendering [S1]. It builds on the idea of combining explicit grid representations with implicit neural fields [S1].

## Sources
[S1] Grid-guided Neural Radiance Fields for Large Urban Scenes, https://doi.org/10.1109/cvpr52729.2023.00802, fetched 2026-10-26
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
