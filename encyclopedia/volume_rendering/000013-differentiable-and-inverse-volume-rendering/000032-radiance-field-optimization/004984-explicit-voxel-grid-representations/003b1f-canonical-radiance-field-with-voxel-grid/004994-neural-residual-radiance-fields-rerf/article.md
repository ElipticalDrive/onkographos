# Neural Residual Radiance Fields (ReRF)

Also called: Residual Radiance Field

## Overview
Neural Residual Radiance Fields (ReRF) is a compact neural representation for dynamic scenes that explicitly models the residual information between adjacent timestamps in the spatial-temporal feature space, using a global coordinate-based tiny MLP as the feature decoder [S1]. Unlike methods relying on a global canonical space which are fragile to large motions, ReRF avoids the global canonical space by exploiting feature similarities between adjacent frames, allowing it to handle large motions without sacrificing quality [S1]. It differs from DeVRF by not relying on a global canonical space, making it robust to large motions or topology changes, by instead exploiting feature similarities between adjacent timestamps [S1].

## How it works
ReRF builds on [Canonical Radiance Field with Voxel Grid](8920-canonical-radiance-field-with-voxel-grid.md) by replacing the heavy MLP with a lightweight one and introducing a sequential residual structure for dynamic scenes.

### Inputs and Outputs
**Inputs:**
- A sequence of RGB images $\{I_t\}_{t=1}^N$ for a dynamic scene.
- Camera poses for each frame.

**Outputs:**
- A global shallow MLP $\Phi$.
- An initial explicit feature grid $f_1$ for the first keyframe.
- A sequence of compact motion grids $\{M_t\}_{t=2}^N$ and sparse residual feature grids $\{r_t\}_{t=2}^N$.

### Data Structures
1.  **Feature Grid ($f$):** A 3D voxel grid storing both density and color features. It is formed by unioning the density grid $V_\sigma$ and color feature grid $V_c$ by appending an additional channel to $V_c$ [S1].
    -   Layout: 3D array.
    -   Channels: Density + Color features.
2.  **Motion Grid ($M_t$):** A low-resolution grid where each entry stores a voxel index offset from the previous frame [S1].
    -   Size: 512 times smaller than the original dense motion field [S1].
    -   Content: Integer voxel offsets.
3.  **Residual Grid ($r_t$):** A sparse grid storing compensation for warping errors and newly observed regions [S1].
    -   Layout: Sparse 3D array (or dense with many zeros).
    -   Content: Feature value deltas.

### Formulas
The radiance field of a static scene using explicit grids is defined as [S1]:
$$ \sigma = \text{interp}(\mathbf{x}, \mathbf{V}_{\sigma}) $$
$$ \mathbf{c} = \Phi(\text{interp}(\mathbf{x}, \mathbf{V}_{c}), \mathbf{d}) $$
where $\text{interp}(\cdot)$ is trilinear interpolation, $\Phi$ is a shallow MLP, $\mathbf{V}_{\sigma}$ is the density grid, $\mathbf{V}_{c}$ is the color feature grid, $\mathbf{x}$ is 3D position, and $\mathbf{d}$ is viewing direction. For simplification, $V_\sigma$ and $V_c$ are unioned into a common feature grid $f$ [S1].

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
1.  **Initialize First Frame:** Train an MLP decoder $\Phi$ and obtain an initial feature grid volume $f_1$ for the first keyframe using an off-the-shelf approach [68] (e.g., Instant-NGP) [S1].
2.  **Estimate Dense Motion:** For each subsequent frame $t$, estimate a dense motion field $D_t$ from the current frame to the previous frame (treating the previous frame as canonical) [S1].
3.  **Generate Compact Motion Grid:** Perform motion pooling on $D_t$ to generate the compact motion grid $M_t$ [S1]:
    -   Split $D_t$ into $8 \times 8 \times 8$ voxel cubes.
    -   Apply average pooling on $D_t$ within each cube to get a mean vector.
    -   Select the voxel $\bar{p}_{t-1}$ that the mean vector points to as $M_t(p_t) = \bar{p}_{t-1}$ [S1].
4.  **Warp Previous Grid:** Warp the previous feature grid $f_{t-1}$ using $M_t$ to obtain the base feature grid $\hat{f}_t$ [S1].
5.  **Optimize Residual Grid:** Optimize the residual grid $r_t$ by backpropagating gradients to $r_t$ while fixing $\hat{f}_t$ and $\Phi$, minimizing the photometric loss and L1 sparsity regularization [S1].
6.  **Recover Current Grid:** Recover the current feature grid $f_t$ by adding the residual to the base grid: $f_t = \hat{f}_t + r_t$ [S1].
7.  **Repeat:** Repeat steps 2-6 for all subsequent frames.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def initialize_first_frame(images_1, poses_1):
    """
    Initialize the first frame using an off-the-shelf method (e.g., Instant-NGP).
    Returns:
        f1: Feature grid for frame 1
        Phi: Global shallow MLP decoder
    """
    # Implementation details depend on the specific off-the-shelf method [68]
    # This step is marked as [G] for implementation specifics not in source
    f1, Phi = train_static_nerf(images_1, poses_1)
    return f1, Phi

def motion_pooling(D_t, kernel_size=8):
    """
    Generate compact motion grid M_t from dense motion field D_t.
    Args:
        D_t: Dense motion field (vector field)
        kernel_size: Size of the pooling cube (default 8)
    Returns:
        M_t: Compact motion grid (integer voxel offsets)
    """
    # Split D_t into cubes of size kernel_size x kernel_size x kernel_size
    # Apply average pooling to get mean vector for each cube
    # Determine the voxel index that the mean vector points to
    # This is a simplified representation of the pooling logic described in [S1]
    M_t = np.zeros_like(D_t, dtype=int) # Placeholder for actual pooling logic
    # Note: The exact implementation of "select the voxel that the mean vector points to"
    # requires discretization logic not fully detailed in the source.
    # [G] Implementation assumes D_t is already discretized to voxel indices.
    return M_t

def warp_grid(f_prev, M_t):
    """
    Warp previous feature grid using motion grid.
    Args:
        f_prev: Feature grid of previous frame
        M_t: Motion grid
    Returns:
        f_hat_t: Base feature grid for current frame
    """
    # f_hat_t(p) = f_prev(p + M_t(p))
    # This involves trilinear interpolation at positions p + M_t(p)
    f_hat_t = np.zeros_like(f_prev)
    for p in all_voxel_indices:
        src_pos = p + M_t[p]
        f_hat_t[p] = trilinear_interpolate(f_prev, src_pos)
    return f_hat_t

def optimize_residual_grid(f_hat_t, Phi, images_t, poses_t, lambda_reg=0.01):
    """
    Optimize residual grid r_t.
    Args:
        f_hat_t: Base feature grid
        Phi: Global MLP
        images_t: Images for current frame
        poses_t: Poses for current frame
        lambda_reg: L1 regularization weight
    Returns:
        r_t: Optimized residual grid
    """
    # Initialize r_t to zeros or small random values
    r_t = np.zeros_like(f_hat_t)
    
    # Optimization loop
    for iteration in range(num_iterations):
        # Sample rays
        rays = sample_rays(images_t, poses_t)
        
        # Compute base features
        base_features = query_grid(f_hat_t, rays.xyz)
        
        # Compute predicted colors
        c_hat = Phi(base_features, rays.directions)
        
        # Compute photometric loss
        loss_photo = np.mean((c_hat - rays.colors) ** 2)
        
        # Compute L1 sparsity regularization
        loss_reg = lambda_reg * np.sum(np.abs(r_t))
        
        # Total loss
        loss = loss_photo + loss_reg
        
        # Backpropagate gradients to r_t only
        # [G] Standard gradient descent step
        gradients = compute_gradients(loss, r_t)
        r_t -= learning_rate * gradients
        
    return r_t

def reconstruct_grid(f_prev, M_t, r_t):
    """
    Reconstruct current feature grid.
    Args:
        f_prev: Previous feature grid
        M_t: Motion grid
        r_t: Residual grid
    Returns:
        f_t: Current feature grid
    """
    f_hat_t = warp_grid(f_prev, M_t)
    f_t = f_hat_t + r_t
    return f_t

def train_rerf(video_sequence):
    """
    Main training loop for ReRF.
    Args:
        video_sequence: List of (images, poses) tuples
    Returns:
        Phi: Global MLP
        f1: Initial feature grid
        motion_grids: List of motion grids
        residual_grids: List of residual grids
    """
    # Step 1: Initialize first frame
    f1, Phi = initialize_first_frame(video_sequence[0].images, video_sequence[0].poses)
    
    motion_grids = []
    residual_grids = []
    f_prev = f1
    
    # Step 2: Sequential training for subsequent frames
    for t in range(1, len(video_sequence)):
        # Estimate dense motion field D_t (from previous to current)
        # [G] Implementation of dense motion estimation not detailed in source
        D_t = estimate_dense_motion(f_prev, video_sequence[t].images, video_sequence[t].poses)
        
        # Generate compact motion grid
        M_t = motion_pooling(D_t)
        motion_grids.append(M_t)
        
        # Warp previous grid
        f_hat_t = warp_grid(f_prev, M_t)
        
        # Optimize residual grid
        r_t = optimize_residual_grid(f_hat_t, Phi, video_sequence[t].images, video_sequence[t].poses)
        residual_grids.append(r_t)
        
        # Recover current grid
        f_t = reconstruct_grid(f_prev, M_t, r_t)
        f_prev = f_t
        
    return Phi, f1, motion_grids, residual_grids
```

### Edge Cases and Pitfalls
-   **Large Motions:** ReRF is designed to handle large motions better than global canonical space methods by using adjacent frame references [S1]. However, if motion exceeds the capacity of the motion grid resolution, warping errors may increase.
-   **Sparsity:** The L1 regularization encourages sparsity in the residual grid. If the scene has high-frequency details or rapid changes, the residual grid may become less sparse, affecting compression efficiency [S1].
-   **Precision:** Trilinear interpolation and motion pooling introduce discretization errors. The choice of kernel size for motion pooling ($8 \times 8 \times 8$) balances compactness and accuracy [S1].

## Subtopics

### ReRF Codec
A compression scheme for ReRF that uses Group of Feature grids (GOF), PCA, 3D DCT, and entropy coding to achieve high compression rates [S1].

### Streamable Player
A playback system that decodes ReRF data on-the-fly for real-time free-viewpoint video rendering [S1].

## Operating characteristics
-   **Time Complexity:** Training is sequential, with each frame depending on the previous one. Motion estimation and residual optimization are performed per frame [S1].
-   **Memory Complexity:** The representation is compact due to the use of motion grids and sparse residual grids. The motion grid is 512 times smaller than the dense motion field [S1].
-   **Measured Performance:** Tested on a dataset with ~74 views at 1920x1080 resolution and 25 fps. Training was performed on a single NVIDIA GeForce RTX3090. Playback was tested on an Intel Core i9-11900 CPU and NVIDIA GeForce RTX3090 GPU [S1].
-   **Strengths:** Handles large motions and topology changes robustly. Achieves three orders of magnitude compression compared to per-frame neural representations [S1].
-   **Limitations:** Sequential training may be slower than parallel methods. Performance depends on the accuracy of motion estimation [S1].
-   **When to Use:** Suitable for long-duration dynamic scenes with large motions where real-time streaming and high compression are required [S1].
-   **When Not to Use:** Not suitable for scenes with extremely rapid, unpredictable motion that exceeds the motion grid's capacity or for static scenes where simpler representations suffice.

## Background
ReRF was proposed by Liao Wang, Minye Wu, and Lan Xu in 2023 [S1]. It addresses the limitations of existing neural rendering techniques for dynamic scenes, which are often restricted to offline rendering or brief sequences with minimal motion [S1]. The method builds on the idea of explicit voxel grids for static scenes [68] and extends it to dynamic scenes by exploiting temporal coherence between adjacent frames, avoiding the fragility of global canonical space methods like DeVRF [S1].

## Sources
[S1] Neural Residual Radiance Fields for Streamably Free-Viewpoint Videos, https://doi.org/10.1109/cvpr52729.2023.00016, fetched 2026-10-26
