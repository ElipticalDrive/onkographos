# Neural Residual Radiance Fields (ReRF)

Also called: Residual Radiance Field

## Overview
Neural Residual Radiance Fields (ReRF) is a compact neural representation for dynamic scenes that explicitly models the residual information between adjacent timestamps in the spatial-temporal feature space, using a global coordinate-based tiny MLP as the feature decoder [S1]. Unlike methods relying on a global canonical space which are fragile to large motions, ReRF avoids the global canonical space by exploiting feature similarities between adjacent frames, allowing it to handle large motions without sacrificing quality [S1]. It differs from DeVRF by not relying on a global canonical space, making it robust to large motions or topology changes, by instead exploiting feature similarities between adjacent timestamps [S1].

## How it works
### Inputs and Outputs
**Inputs:**
- A sequence of RGB images $\{I_t\}_{t=1}^N$ for a dynamic scene [S1].
- Camera poses for each frame [S1].

**Outputs:**
- A global shallow MLP $\Phi$ [S1].
- An initial explicit feature grid $f_1$ for the first keyframe [S1].
- A sequence of compact motion grids $\{M_t\}_{t=2}^N$ and sparse residual feature grids $\{r_t\}_{t=2}^N$ [S1].

### Data Structures
1.  **Feature Grid ($f$):** A 3D voxel grid storing both density and color features [S1]. It is formed by unioning the density grid $V_\sigma$ and color feature grid $V_c$ by appending an additional channel to $V_c$ [S1].
    -   Layout: 3D array [S1].
    -   Channels: Density + Color features [S1].
2.  **Motion Grid ($M_t$):** A low-resolution grid where each entry stores a voxel index offset from the previous frame [S1].
    -   Size: 512 times smaller than the original dense motion field [S1].
    -   Content: Integer voxel offsets [S1].
3.  **Residual Grid ($r_t$):** A sparse grid storing compensation for warping errors and newly observed regions [S1].
    -   Layout: Sparse 3D array (or dense with many zeros) [S1].
    -   Content: Feature value deltas [S1].

### Formulas
The radiance field of a static scene using explicit grids is defined as [S1]:
$$ \sigma = \text{interp}(\mathbf{x}, \mathbf{V}_{\sigma}) $$
$$ \mathbf{c} = \Phi(\text{interp}(\mathbf{x}, \mathbf{V}_{c}), \mathbf{d}) $$
where $\text{interp}(\cdot)$ is trilinear interpolation, $\Phi$ is a shallow MLP, $\mathbf{V}_{\sigma}$ is the density grid, $\mathbf{V}_{c}$ is the color feature grid, $\mathbf{x}$ is 3D position, and $\mathbf{d}$ is viewing direction [S1]. For simplification, $V_\sigma$ and $V_c$ are unioned into a common feature grid $f$ [S1].

The base feature grid $\hat{f}_t$ for frame $t$ is obtained by warping the previous frame's feature grid $f_{t-1}$ using the motion grid $M_t$ [S1]:
$$ \hat{\mathbf{f}}_{t}(\mathbf{p}) = \mathbf{f}_{t-1}(\mathbf{p} + \mathbf{M}_t(\mathbf{p})) $$
where $\mathbf{p}$ is the voxel index [S1].

The total loss function for learning the residual grid $r_t$ is [S1]:
$$ \mathcal{L}_{total} = \sum_{\mathbf{l} \in \mathbb{L}}\|\mathbf{c}(\mathbf{l})-\hat{\mathbf{c}}(\mathbf{l})\|^2 + \lambda \|\mathbf{r}_t\|_1 $$
where $\mathbb{L}$ is the set of training pixel rays, $\mathbf{c}(\mathbf{l})$ and $\hat{\mathbf{c}}(\mathbf{l})$ are ground truth and predicted colors, and $\lambda = 0.01$ [S1].

### Parameters
-   **Motion pooling kernel size:** $8 \times 8 \times 8$ voxels [S1].
-   **Regularization weight $\lambda$:** 0.01 [S1].
-   **GOF (Group of Feature grids) length:** 20 frames [S1].

### Steps
1.  **Initialize First Frame:** Train an MLP decoder $\Phi$ and obtain an initial feature grid volume $f_1$ for the first keyframe using an off-the-shelf approach [68] [S1].
2.  **Estimate Dense Motion:** For each subsequent frame $t$, estimate a dense motion field $D_t$ from the current frame to the previous frame (treating the previous frame as canonical) [S1].
3.  **Generate Compact Motion Grid:** Perform motion pooling on $D_t$ to generate the compact motion grid $M_t$ [S1]:
    -   Split $D_t$ into $8 \times 8 \times 8$ voxel cubes [S1].
    -   Apply average pooling on $D_t$ within each cube to get a mean vector [S1].
    -   Select the voxel $\bar{p}_{t-1}$ that the mean vector points to as $M_t(p_t) = \bar{p}_{t-1}$ [S1].
4.  **Warp Previous Grid:** Warp the previous feature grid $f_{t-1}$ using $M_t$ to obtain the base feature grid $\hat{f}_t$ [S1].
5.  **Optimize Residual Grid:** Optimize the residual grid $r_t$ by backpropagating gradients to $r_t$ while fixing $\hat{f}_t$ and $\Phi$, minimizing the photometric loss and L1 sparsity regularization [S1].
6.  **Recover Current Grid:** Recover the current feature grid $f_t$ by adding the residual to the base grid: $f_t = \hat{f}_t + r_t$ [S1].
7.  **Repeat:** Repeat steps 2-6 for all subsequent frames [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def motion_pooling(D_t, kernel_size=8):
    # Split D_t into 8x8x8 voxel cubes.
    # Apply average pooling on D_t within each cube to get a mean vector.
    # Select the voxel index that the mean vector points to as M_t.
    # [S1] The sources do not provide the exact discretization logic for mapping mean vectors to voxel indices.
    pass

def warp_grid(f_prev, M_t):
    # f_hat_t(p) = f_prev(p + M_t(p))
    # Trilinear interpolation at positions p + M_t(p)
    # [G] Standard trilinear interpolation on a 3D grid.
    pass

def optimize_residual_grid(f_hat_t, Phi, images_t, poses_t, lambda_reg=0.01):
    # Optimize residual grid r_t by backpropagating gradients to r_t
    # while fixing f_hat_t and Phi, minimizing photometric loss and L1 sparsity regularization.
    # [S1] The sources do not provide the exact optimization loop details.
    pass

def reconstruct_grid(f_prev, M_t, r_t):
    f_hat_t = warp_grid(f_prev, M_t)
    f_t = f_hat_t + r_t
    return f_t

def train_rerf(video_sequence):
    # Step 1: Initialize first frame with complete explicit feature grid f1 and global MLP Phi
    # using off-the-shelf approach [68]. [S1]
    f1, Phi = None, None # [S1] The sources do not provide the exact initialization routine.

    motion_grids = []
    residual_grids = []
    f_prev = f1

    # Step 2: Sequential training for subsequent frames
    for t in range(1, len(video_sequence)):
        # Estimate dense motion field D_t from current frame to previous frame. [S1]
        D_t = None # [S1] The sources do not provide the exact dense motion estimation routine.

        # Generate compact motion grid M_t via motion pooling. [S1]
        M_t = motion_pooling(D_t)
        motion_grids.append(M_t)

        # Warp previous feature grid f_{t-1} using M_t to obtain base feature grid f_hat_t. [S1]
        f_hat_t = warp_grid(f_prev, M_t)

        # Optimize residual grid r_t. [S1]
        r_t = optimize_residual_grid(f_hat_t, Phi, video_sequence[t].images, video_sequence[t].poses)
        residual_grids.append(r_t)

        # Recover current feature grid f_t. [S1]
        f_t = reconstruct_grid(f_prev, M_t, r_t)
        f_prev = f_t

    return Phi, f1, motion_grids, residual_grids
```

### Edge Cases and Pitfalls
-   **Large Motions:** ReRF is designed to handle large motions better than global canonical space methods by using adjacent frame references [S1].
-   **Sparsity:** The L1 regularization encourages sparsity in the residual grid [S1].
-   **Precision:** Trilinear interpolation and motion pooling introduce discretization errors [S1]. The choice of kernel size for motion pooling ($8 \times 8 \times 8$) balances compactness and accuracy [S1].

## Subtopics

### ReRF Codec
A compression scheme for ReRF that uses Group of Feature grids (GOF), PCA, 3D DCT, and entropy coding to achieve high compression rates [S1].

### Streamable Player
A playback system that decodes ReRF data on-the-fly for real-time free-viewpoint video rendering [S1].

## Operating characteristics
-   **Time Complexity:** Training is sequential, with each frame depending on the previous one [S1]. Motion estimation and residual optimization are performed per frame [S1].
-   **Memory Complexity:** The representation is compact due to the use of motion grids and sparse residual grids [S1]. The motion grid is 512 times smaller than the dense motion field [S1].
-   **Measured Performance:** Tested on a dataset with ~74 views at 1920x1080 resolution and 25 fps [S1]. Training was performed on a single NVIDIA GeForce RTX3090 [S1]. Playback was tested on an Intel Core i9-11900 CPU and NVIDIA GeForce RTX3090 GPU [S1].
-   **Strengths:** Handles large motions and topology changes robustly [S1]. Achieves three orders of magnitude compression compared to per-frame neural representations [S1].
-   **Limitations:** Sequential training may be slower than parallel methods [S1]. Performance depends on the accuracy of motion estimation [S1].
-   **When to Use:** Suitable for long-duration dynamic scenes with large motions where real-time streaming and high compression are required [S1].
-   **When Not to Use:** Not suitable for scenes with extremely rapid, unpredictable motion that exceeds the motion grid's capacity or for static scenes where simpler representations suffice [S1].

## Background
ReRF was proposed by Liao Wang, Minye Wu, and Lan Xu in 2023 [S1]. It addresses the limitations of existing neural rendering techniques for dynamic scenes, which are often restricted to offline rendering or brief sequences with minimal motion [S1]. The method builds on the idea of explicit voxel grids for static scenes [68] and extends it to dynamic scenes by exploiting temporal coherence between adjacent frames, avoiding the fragility of global canonical space methods like DeVRF [S1].

## Sources
[S1] Neural Residual Radiance Fields for Streamably Free-Viewpoint Videos, https://doi.org/10.1109/cvpr52729.2023.00016, fetched 2026-10-26
