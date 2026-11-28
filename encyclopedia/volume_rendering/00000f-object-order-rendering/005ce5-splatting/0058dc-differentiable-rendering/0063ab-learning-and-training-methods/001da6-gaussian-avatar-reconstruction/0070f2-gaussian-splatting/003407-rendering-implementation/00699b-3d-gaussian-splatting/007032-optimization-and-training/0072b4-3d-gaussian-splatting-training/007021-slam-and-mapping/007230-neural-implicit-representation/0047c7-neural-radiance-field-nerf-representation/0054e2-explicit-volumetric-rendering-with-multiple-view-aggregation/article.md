# Explicit Volumetric Rendering with Multiple View Aggregation

Also called: Pooled Latent Volume Rendering, RGBα Volume Rendering

## Overview
This method models scene geometry and appearance as an explicit RGBα volume to amortize rendering across multiple views, solving the computational inefficiency of NeRF-like methods where per-ray sampling is independent and not reusable across different viewing angles [S1]. Unlike its siblings [Scene NeRF](12484-scene-nerf.md), [NeRF-OSR](14477-nerf-osr.md), and [Dynamic Neural Radiance Fields](15318-dynamic-neural-radiance-fields.md), which typically rely on implicit field evaluations or sparse ray sampling, this approach generates a dense volumetric representation that enables real-time rendering and the use of structural losses like SSIM during training [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $n$ source views $S = \{I_1, \dots, I_n\}$ with corresponding camera poses $P = \{P_1, \dots, P_n\}$ and intrinsic parameters [S1].
*   **Outputs:** An RGBα volume $V \in \mathbb{R}^{4 \times d_s \times h \times w}$ and rendered target images $\hat{C}$ [S1].

**Data Structures:**
*   **Latent Volume ($Z_i$):** An intermediate volumetric representation for each source view $i$, processed by a 3D U-Net [S1].
*   **RGBα Volume ($V$):** A 4D tensor with dimensions $4 \times d_s \times h \times w$, where the first 3 channels encode color $\mathbf{c}$ and the last channel encodes alpha $\alpha$ [S1].

**Numbered Steps:**
1.  **Encode Source Views:** For each source view $i \in \{1, \dots, n\}$, encode the image $I_i$ into a latent volume $Z_i$ using the encoding process from the parent article [S1].
2.  **Select Origin:** Select an arbitrary source view $i^*$ from the set of source views to serve as the origin of the coordinate system [S1].
3.  **Align Volumes:** Align each latent volume $Z_i$ to the origin $i^*$ using the relative transformation $P_{i, i^*}$ between the pose $P_i$ of view $i$ and the pose $P_{i^*}$ of the origin [S1].
4.  **Pool Volumes:** Compute the pooled volumetric latent $\bar{Z}$ by taking the mean of the aligned latent volumes across all $n$ views:
    $$ \bar{\mathbf{Z}} = \frac{1}{n} \sum_{i=1}^{n} \mathcal{T}(\mathbf{P}_{i, i^*}, \mathbf{Z}_i) $$
    [S1]
5.  **Generate RGBα Volume:** Feed the pooled latent $\bar{Z}$ into a 3D U-Net to generate the explicit RGBα volume $V$ [S1].
6.  **Perspective Deformation:** Apply perspective deformation to the viewing frustum using the intrinsic camera parameters via inverse warping and trilinear sampling [S1].
7.  **Render:** For a given pixel location $(i, j)$, calculate the expected color $\hat{C}_{i,j}$ by integrating along the depth samples $k=1 \dots d_s$:
    $$ \hat{C}_{i,j} = \sum_{k=1}^{d_s} T_{i,j}^k \alpha_{i,j}^k \mathbf{c}_{i,j}^k $$
    where the transmittance $T_{i,j}^k$ is defined as:
    $$ T_{i,j}^k = \prod_{m=1}^{k-1} (1-\alpha_{i,j}^m) $$
    Here, $\mathbf{c}_{i,j}^k$ is the color value from the first 3 channels of $V$ at sample $k$, and $\alpha_{i,j}^k$ is the alpha value from the last channel of $V$ at sample $k$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def render_explicit_volume(source_views, poses, intrinsics, origin_index):
    n = len(source_views)
    aligned_latents = []
    for i in range(n):
        # Step 1: Encode source view into latent volume Z_i
        Z_i = conv_encoder(source_views[i])
        # Step 3: Compute relative transformation P_i,i* = P_i * P_i*^-1
        P_rel = poses[i] @ np.linalg.inv(poses[origin_index])
        # Step 3: Align latent volume to origin coordinate system
        Z_aligned = affine_transform_3d(Z_i, P_rel)
        aligned_latents.append(Z_aligned)
    # Step 4: Pool aligned volumes by taking the mean
    pooled_latent = sum(aligned_latents) / n
    # Step 5: Feed pooled latent to 3D U-Net to generate RGBα volume V
    V = u3d_forward(pooled_latent)
    # Step 6 & 7: Perspective deformation and rendering
    return render_with_deformation(V, intrinsics)

def conv_encoder(image):
    # Standard 2D convolutional network mapping image to 3D latent volume [G]
    return network(image)

def affine_transform_3d(volume, transform_matrix):
    # 3D grid sampling with affine transformation [G]
    grid = generate_3d_grid(volume.shape)
    transformed_coords = (transform_matrix @ grid.T).T
    return trilinear_interpolation(volume, transformed_coords)

def u3d_forward(latent):
    # 3D U-Net forward pass [G]
    return network(latent)

def render_with_deformation(V, intrinsics):
    h, w = V.shape[2], V.shape[3]
    ds = V.shape[1]
    image = np.zeros((h, w, 3))
    for j in range(w):
        for i in range(h):
            T = 1.0
            color = np.zeros(3)
            for k in range(1, ds + 1):
                alpha = V[3, k-1, i, j]
                c = V[0:3, k-1, i, j]
                transmittance = T
                T *= (1.0 - alpha)
                color += transmittance * alpha * c
            image[i, j] = color
    # Step 6: Apply perspective deformation using inverse warping [S1]
    return inverse_warp(image, intrinsics)
```

**Parameters:**
*   $\lambda$: Weight for the SSIM loss in the training objective [S1].
*   $d_s$: Number of depth samples [S1].

**Edge Cases and Pitfalls:**
*   **Coordinate System Alignment:** The choice of origin view $i^*$ is arbitrary, but all latent volumes must be consistently aligned to this origin to ensure correct pooling [S1].
*   **Precision:** Trilinear sampling is used for perspective deformation; care must be taken with boundary conditions and interpolation artifacts [S1].
*   **Degenerate Input:** The method assumes a valid set of source views with known poses [S1].

## Subtopics

### [Scene NeRF](12484-scene-nerf.md)
Sibling method.

### [NeRF-OSR](14477-nerf-osr.md)
Sibling method.

### [Dynamic Neural Radiance Fields](15318-dynamic-neural-radiance-fields.md)
Sibling method.

### [Neural Network Pruning for NeRF](17703-neural-network-pruning-for-nerf.md)
Sibling method.

### [Scene Auto-Encoder](17789-scene-auto-encoder.md)
Sibling method.

### [Bi-level Neural Volume Fusion](19888-bi-level-neural-volume-fusion.md)
Sibling method.

### [SLAM and Pose Optimization](19899-slam-and-pose-optimization.md)
Sibling method.

## Operating characteristics
*   **Time Complexity:** Rendering is amortized across views [S1]. Per-view rendering time is 0.0178s (56 FPS) on an NVIDIA Tesla V100 [S1].
*   **Memory Complexity:** Requires storing the RGBα volume $V$ and latent volumes $Z_i$ [S1]. Exact memory footprint not given in the sources.
*   **Measured Performance:** Rendering 250 views takes 1.022s (245 FPS), a 400x speedup over pixelNeRF's 474.8606s (0.5 FPS) [S1]. Per-view, it is 100x faster than pixelNeRF's 1.9047s [S1].
*   **Strengths:** Real-time rendering capability; supports structural losses like SSIM during training because it renders full images rather than sparse rays [S1].
*   **Limitations:** Requires explicit volumetric storage [S1].
*   **When to Use:** When real-time novel view synthesis is required and multiple source views are available [S1]. Evaluated on ShapeNet chairs and cars [S1].
*   **When Not to Use:** When memory is extremely constrained and explicit volumes cannot be stored, or when only a single view is available for initialization [G].

## Background
The method was introduced in "Fast and Explicit Neural View Synthesis" [S1]. It addresses the limitation of NeRF-like methods where rendering cannot be amortized across views because each ray integral is independent [S1]. By using an explicit RGBα volume, the method allows for efficient rendering and the application of structural losses, which are not directly applicable to NeRF-like methods due to their sparse ray sampling [S1]. It has been evaluated on ShapeNet chairs and cars categories, demonstrating competitive visual quality and significant speed improvements over baselines like pixelNeRF [S1].

## Sources
[S1] Fast and Explicit Neural View Synthesis, https://doi.org/10.1109/wacv51458.2022.00009, fetched 2026-10-30
