# Compact 3D Gaussian Representation (C3DGS)

## Overview
Compact 3D Gaussian Representation (C3DGS) is a method for reducing the memory footprint and storage requirements of 3D Gaussian Splatting (3DGS) by exploiting spatial redundancy in scene attributes [S1]. It addresses the high memory cost of storing per-Gaussian attributes by replacing dense per-point storage with a grid-based neural field for view-dependent colors and a codebook for geometric parameters [S1]. Unlike standard 3DGS, which treats each Gaussian as an independent primitive, C3DGS shares color information across neighboring points and quantizes geometry, achieving over 25× storage compression while maintaining or improving reconstruction quality [S1].

## How it works
C3DGS operates in two main phases: Gaussian reduction via volume-based masking, and attribute compression via grid and codebook representations [S1].

**Inputs:**
*   Initial set of 3D Gaussians $\mathcal{G} = \{G_i\}_{i=1}^N$ from a standard 3DGS training process, where each $G_i$ has position $\mathbf{p}_i$, opacity $\alpha_i$, scale $\mathbf{s}_i$, rotation $\mathbf{q}_i$, and SH coefficients $\mathbf{c}_i$ [S1].
*   Training images and camera poses.

**Outputs:**
*   A compact representation consisting of:
    *   A reduced set of active Gaussian indices.
    *   A hash-based grid neural field for color.
    *   A codebook for geometry (scale and rotation).
    *   Per-Gaussian indices pointing to the grid and codebook.

**Data Structures:**

1.  **Volume Mask:** A learnable scalar mask $m_i \in [0, 1]$ for each Gaussian $G_i$. If $m_i < \tau$ (threshold), the Gaussian is pruned [S1].
2.  **Hash Grid:** A multi-resolution hash grid (Instant NGP style) mapping 3D positions to feature vectors [S1]. Let $H$ denote this grid.
3.  **Geometry Codebook:** A set of $K$ learned geometric prototypes $\mathcal{C} = \{\mathbf{z}_k\}_{k=1}^K$, where each $\mathbf{z}_k$ represents a pair of $(\mathbf{s}, \mathbf{q})$ [S1].
4.  **Indices:**
    *   $idx^{color}_i$: Index into the hash grid features (implicitly determined by position $\mathbf{p}_i$).
    *   $idx^{geo}_i \in \{1, \dots, K\}$: Codebook index for Gaussian $i$ [S1].

**Numbered Steps:**

1.  **Gaussian Reduction (Volume-based Masking):**
    During training, introduce a learnable mask $m_i$ for each Gaussian. Optimize $m_i$ alongside other parameters. Gaussians with $m_i$ below a threshold are removed from the rendering pipeline, reducing $N$ [S1]. This step identifies and removes non-essential Gaussians with minimal impact on performance [S1].

2.  **Color Compression (Hash Grid):**
    Instead of storing SH coefficients $\mathbf{c}_i$ for each Gaussian, query the hash grid $H$ at position $\mathbf{p}_i$ to retrieve a feature vector. This feature vector is passed through a small MLP to predict the view-dependent color [S1]. This exploits spatial redundancy where neighboring Gaussians share similar colors [S1].

3.  **Geometry Compression (Codebook):**
    Replace the explicit storage of scale $\mathbf{s}_i$ and rotation $\mathbf{q}_i$ with a codebook index $idx^{geo}_i$. The actual geometry is retrieved from the codebook $\mathcal{C}$ [S1]. The codebook learns common geometric patterns across the scene [S1].

4.  **Training Optimization:**
    The total loss includes the standard rendering loss (e.g., L1, D-SSIM) plus regularization terms for the mask and codebook assignments. The hash grid and codebook are updated via backpropagation [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# C3DGS Training Step
def c3dgs_step(gaussians, images, cameras, hash_grid, codebook, mask):
    """
    gaussians: List of Gaussian primitives with positions p_i, opacities alpha_i
    hash_grid: Instant NGP hash grid structure
    codebook: Tensor of shape [K, 6] representing K prototypes of (scale, rotation)
    mask: Tensor of shape [N] with learnable mask values m_i
    images: Ground truth images
    cameras: Camera parameters
    """
    
    # 1. Prune Gaussians based on mask
    active_indices = [i for i, m in enumerate(mask) if m > 0.5] # Threshold [G]
    active_gaussians = [gaussians[i] for i in active_indices]
    
    # 2. Retrieve compressed attributes
    # Color: Query hash grid at position p_i
    # Note: Instant NGP query returns features, then MLP predicts color
    colors = []
    for g in active_gaussians:
        features = hash_grid.query(g.position)
        color = color_mlp(features) # MLP defined in C3DGS paper [S1]
        colors.append(color)
        
    # Geometry: Retrieve from codebook
    # Assume codebook indices are stored or optimized
    # For simplicity, assume we have pre-computed indices idx_geo
    # In practice, idx_geo is learned or assigned via nearest neighbor
    geometries = []
    for g in active_gaussians:
        idx = g.codebook_index
        geo = codebook[idx] # Shape: [6] -> (scale, rotation)
        geometries.append(geo)
        
    # 3. Render
    rendered_images = render(active_gaussians, geometries, colors, cameras)
    
    # 4. Compute Loss
    loss = compute_loss(rendered_images, images)
    
    # 5. Backpropagate
    loss.backward()
    
    return loss
```

**Parameters:**
*   **Grid Type:** Hash-based grid (Instant NGP) [S1].
*   **Codebook Size:** Small, specific value not given in sources [S1].
*   **Compression Ratio:** ~15× average, up to 28× on Deep Blending dataset [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Hash grid queries may introduce interpolation artifacts if resolution is too low [S1].
*   **Sparsity:** In sparse scenes, the hash grid may not capture local continuity as effectively as in dense scenes [S1].
*   **Masking Threshold:** Choosing the threshold for mask pruning is critical; too aggressive pruning leads to quality loss [S1].

## Subtopics

### [Optimized Minimal 3D Gaussian Splatting (OMG)](13978-optimized-minimal-3d-gaussian-splatting-omg.md)
OMG extends C3DGS by retaining per-Gaussian parameterization for geometry to handle sparsity, while using a small neural field and Sub-Vector Quantization (SVQ) for appearance features. It addresses C3DGS's weakness in sparse settings where neighboring Gaussians are further apart.

## Operating characteristics
*   **Storage:** Achieves over 25× compression compared to standard 3DGS [S1]. On the Deep Blending dataset, it achieves over 28× enhancement in storage efficiency [S1].
*   **Speed:** Nearly 40% increase in rendering speed on Deep Blending dataset [S1]. General rendering speed is enhanced compared to 3DGS [S1].
*   **Quality:** Maintains or improves reconstruction quality (PSNR) compared to 3DGS [S1].
*   **Scalability:** Computational complexity is linearly proportional to the number of Gaussians, so reducing Gaussians directly speeds up rendering [S1].
*   **Limitations:** Relies on spatial redundancy; may be less effective in highly sparse or unstructured scenes [S1].

## Background
C3DGS was proposed to address the critical memory and storage issues in 3DGS, which often require >1GB for large real-world scenes [S1]. It builds on the observation that current densification algorithms produce redundant Gaussians and that neighboring Gaussians share similar attributes [S1]. The method uses Instant NGP for the hash grid [S1].

## Sources
[S1] Compact 3D Gaussian Representation for Radiance Field, https://doi.org/10.1109/cvpr52733.2024.02052, fetched 2026-10-26
