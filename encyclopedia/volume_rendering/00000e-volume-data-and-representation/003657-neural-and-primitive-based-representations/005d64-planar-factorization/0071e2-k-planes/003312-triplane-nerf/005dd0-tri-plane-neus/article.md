# Tri-Plane NeuS

Also called: TP-NeuS (Baseline)

## Overview
Tri-Plane NeuS is a neural surface reconstruction method that combines the TriPlane NeRF representation with the NeuS implicit surface formulation. It models the Signed Distance Function (SDF) using tri-plane features and an MLP, solving the problem of high-fidelity 3D surface recovery from multi-view images. It differs from its parent [TriPlane NeRF](6932-triplane-nerf.md) by optimizing for an SDF rather than volume density, and from [NeuS](link-to-neus) by using explicit tri-plane features instead of a purely implicit MLP for spatial encoding.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ images with known camera poses.
*   **Output:** A reconstructed 3D mesh (surface) and texture.

**Data Structures:**
*   **Tri-Planes ($T$):** Three 2D feature planes $T_{xy}, T_{yz}, T_{zx}$, each of size $H \times W \times C$.
*   **MLP:** A multi-layer perceptron mapping concatenated features to SDF and opacity.

**Method Steps:**
1.  **Feature Query:** For a 3D point $\mathbf{x} = (x, y, z)$, project it onto the three axis-aligned planes to get 2D coordinates. Sample features from $T_{xy}, T_{yz}, T_{zx}$ at these coordinates.
2.  **Concatenation:** Concatenate the sampled features from the three planes: $\mathbf{f} = [T_{xy}(\mathbf{x}), T_{yz}(\mathbf{x}), T_{zx}(\mathbf{x})]$.
3.  **SDF Prediction:** Pass $\mathbf{f}$ through the MLP to predict the SDF value $S(\mathbf{x})$.
    $$ S(\mathbf{x}) = \text{MLP}([\mathbf{T}(\mathbf{x})]) $$
    where $[\mathbf{T}(\mathbf{x})]$ denotes the concatenated tri-plane features.
4.  **Rendering:** Use volumetric rendering with the SDF to render images and compute photometric loss.
5.  **Surface Extraction:** Extract the mesh using the marching cubes algorithm on the learned SDF field.

**Parameters:**
*   Tri-plane resolution: Not given in the sources.
*   Feature dimension $C$: Not given in the sources.
*   MLP architecture: 3-layer MLP (implied by initialization section).

**Edge Cases and Pitfalls:**
*   **Foreground/Background Sticking:** Without geometric initialization, the SDF may fail to separate foreground and background, leading to incorrect topology (e.g., on complex geometry like a belly) [S1].
*   **High-Frequency Artifacts:** Standard MLP initialization combined with random tri-plane initialization can introduce high-frequency artifacts on the surface [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def predict_sdf(xyz, tri_planes, mlp):
    """
    Predicts SDF for a batch of 3D points using Tri-Plane NeuS.
    
    Args:
        xyz: Tensor of shape (N, 3), 3D coordinates.
        tri_planes: Dictionary containing 'xy', 'yz', 'zx' tensors of shape (H, W, C).
        mlp: Neural network module.
        
    Returns:
        sdf: Tensor of shape (N, 1), predicted SDF values.
    """
    # 1. Project 3D points to 2D planes
    # xy plane: (x, y)
    xy_coords = xyz[:, :2]
    # yz plane: (y, z)
    yz_coords = xyz[:, 1:3]
    # zx plane: (z, x)
    zx_coords = torch.stack((xyz[:, 2], xyz[:, 0]), dim=1)
    
    # 2. Sample features from tri-planes
    # Assuming a helper function 'sample_features' that performs bilinear interpolation
    # at normalized coordinates [-1, 1]
    feat_xy = sample_features(tri_planes['xy'], xy_coords)
    feat_yz = sample_features(tri_planes['yz'], yz_coords)
    feat_zx = sample_features(tri_planes['zx'], zx_coords)
    
    # 3. Concatenate features
    # feat_xy, feat_yz, feat_zx are shape (N, C)
    features = torch.cat([feat_xy, feat_yz, feat_zx], dim=1)
    
    # 4. Pass through MLP
    sdf = mlp(features)
    
    return sdf
```

## Subtopics

### Geometric Initialization for Tri-Planes
A method to initialize tri-plane features to prevent foreground/background sticking and high-frequency artifacts. It is compared against random noise initialization and standard MLP geometric initialization. On the DTU dataset, the mean Chamfer Distance for "No Geo Init" is 1.86, "MLP Geo Init" is 1.34, and "Our Geo Init" is 1.05 [S1]. "No Geo Init" (random noise $N(0,1)$ for tri-planes, standard init for MLP) causes foreground/background sticking and inconsistent reconstruction on complex geometry [S1]. "MLP Geo Init" (random noise for tri-planes, standard init for MLP) results in high-frequency detail artifacts on the generated surface model [S1]. "Our Geo Init" leads to more consistent surface reconstruction results [S1].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
On the DTU dataset, the mean Chamfer Distance for the baseline Tri-Plane NeuS (with geometric initialization) is 1.05 [S1].

**Strengths and Limitations:**
*   **Strengths:** Prevents foreground/background sticking and reduces high-frequency artifacts compared to naive initialization [S1].
*   **Limitations:** The baseline model relies on positional encoding for fine details when extended (TP-NeuS + PE), suggesting the tri-planes alone may lack high-frequency capacity without additional encoding [S1].

**When to Use:**
Use when a baseline neural surface reconstruction method with explicit tri-plane features is required, particularly when geometric initialization is critical for stable training on complex geometries.

## Background
Tri-Plane NeuS is the baseline model described in PET-NeuS, which uses tri-planes and MLPs to model the SDF [S1]. It builds on the TriPlane NeRF representation [6932-triplane-nerf.md] and the NeuS implicit surface formulation. The method was introduced in PET-NeuS: Positional Encoding Tri-Planes for Neural Surfaces [S1].

## Sources
[S1] PET-NeuS: Positional Encoding Tri-Planes for Neural Surfaces, http://arxiv.org/abs/2305.05594v1, fetched 2026-10-27
