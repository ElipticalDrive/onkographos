# Learnable Pose Refinement via Local SE(3) Warp

Also called: Embedding-Based Warp MLP, Local SE(3) Pose Correction

## Overview
This method refines coarse camera poses by learning a per-camera correction in the SE(3) Lie algebra space, conditioned on 2D image coordinates and a learnable embedding [S2, unofficial]. It addresses the problem of pose inaccuracies in sparse or noisy initial estimates by jointly optimizing the pose corrections alongside the radiance field, avoiding the instability of direct global pose optimization [S1][S2, unofficial]. Unlike methods that rely solely on geometric consistency or robust PnP variants [S1], this approach uses a differentiable neural network to predict local SE(3) warps, allowing for end-to-end training with photometric and multi-view losses.

## How it works
The method optimizes camera poses by predicting a local SE(3) transformation for each view, which is composed with the initial pose estimate [S2, unofficial]. The refinement is driven by a neural network (Warp MLP) that takes 2D pixel coordinates and a per-camera embedding as input [S2, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Initial camera poses $P_{init} \in SE(3)^{N}$ (rotation $R_{init}$, translation $t_{init}$) [S1][S2, unofficial].
    *   Inverse intrinsics $K^{-1}$ [S2, unofficial].
    *   Ray indices or pixel coordinates [S2, unofficial].
    *   Per-camera embeddings $E \in \mathbb{R}^{N \times d_{emb}}$ [S2, unofficial].
*   **Outputs:**
    *   Refined camera poses $P_{refined} \in SE(3)^{N}$ [S2, unofficial].
    *   Gradients for pose embeddings, MLP weights, and initial pose parameters [S1][S2, unofficial].

### Data Structures
The implementation relies on the following structures [S2, unofficial]:

1.  **Warp Embedding:**
    *   Type: `torch.nn.Embedding`
    *   Layout: `weight` tensor of shape `(num_cameras, embedding_dim)`.
    *   Purpose: Stores a learnable vector $e_i$ for each camera $i$ to capture view-specific pose biases.

2.  **Warp MLP:**
    *   Type: `localWarp` (custom neural network module).
    *   Input: Concatenation of 2D coordinates $(u, v)$ and embedding $e_i$.
    *   Output: 6-dimensional vector representing the SE(3) Lie algebra $\xi \in \mathbb{R}^6$.

3.  **Camera Coordinates Grid:**
    *   Dynamically gathered 3D coordinates of camera centers or ray origins, used to derive 2D projections [S2, unofficial].

### Algorithm Steps

1.  **Initialization:**
    *   Initialize initial poses $R_0, t_0$ from a coarse estimator (e.g., COLMAP) [S1].
    *   Initialize per-camera embeddings $E$ randomly or with zeros [G].
    *   Initialize Warp MLP weights [S2, unofficial].

2.  **Ray Sampling:**
    *   For each optimization step, sample $N_{rays}$ pixel rays per image.
    *   Typical values: 1024 rays for DTU/Replica, 2048 rays for LLFF [S1].

3.  **Coordinate Extraction:**
    *   Gather 3D camera coordinates for the sampled rays: $C_{3D} = \text{gather\_camera\_cords\_grid\_3D}(opt, batch\_size, K^{-1}, ray\_idx)$ [S2, unofficial].
    *   Extract 2D coordinates from the 3D grid: $C_{2D} = C_{3D}[:, :, :2]$ [S2, unofficial].

4.  **Embedding Retrieval:**
    *   Retrieve the embedding for the current camera $i$: $e_i = E[i]$.
    *   Expand $e_i$ to match the number of sampled rays: $E_{expanded} = \text{expand}(e_i, (-1, N_{rays}, -1))$ [S2, unofficial].

5.  **Local SE(3) Prediction:**
    *   Concatenate 2D coordinates and embedding: $Input = \text{concat}(C_{2D}, E_{expanded}, dim=-1)$.
    *   Pass through Warp MLP: $\xi = \text{WarpMLP}(Input)$, where $\xi \in \mathbb{R}^{N_{rays} \times 6}$ [S2, unofficial].

6.  **SE(3) Matrix Conversion:**
    *   Convert the Lie algebra vector $\xi$ to an SE(3) matrix $M_{local}$ using the source's `se3_to_SE3` function [S2, unofficial].

7.  **Pose Composition:**
    *   Compose the local refinement with the initial pose $P_{init}$: $P_{refined} = \text{compose}([M_{local}, P_{init}])$ [S2, unofficial].

8.  **Rotation Matrix Reconstruction (Alternative/Specific Variant):**
    *   In some implementations (e.g., SPARF), the rotation is updated using a Gram-Schmidt-like process [S1].
    *   Input: 4D vector $r = [a_1^T, a_2^T]^T$.
    *   Compute basis vectors:
        $$ b_1 = N(a_1) $$
        $$ b_2 = N(a_2 - (b_1 \cdot a_2)b_1) $$
        $$ b_3 = b_1 \times b_2 $$
        where $N(\cdot)$ denotes L2 norm [S1].
    *   Construct Rotation Matrix:
        $$ R = [b_1, b_2, b_3] $$
    *   Update parameters:
        $$ \hat{R}_{w2c} = f(\hat{r}_{w2c} + \Delta r) $$
        $$ \hat{t}_{w2c} = t_{w2c} + \Delta t $$
        where $f$ is the function defined above [S1].

### Parameters and Hyperparameters

*   **Learning Rates (Adam Optimizer):**
    *   **Network Weights:**
        *   DTU/Replica: Initial $5 \times 10^{-4}$, decaying to $1 \times 10^{-4}$ [S1].
        *   LLFF: Initial $1 \times 10^{-3}$, decaying to $1 \times 10^{-4}$ [S1].
    *   **Camera Poses:**
        *   DTU/Replica: Initial $1 \times 10^{-3}$, decaying to $1 \times 10^{-4}$ [S1].
        *   LLFF: Initial $3 \times 10^{-3}$, decaying to $1 \times 10^{-5}$ [S1].
*   **Image Resolution:**
    *   DTU: $300 \times 400$ [S1].
    *   Replica: $340 \times 600$ [S1].
    *   LLFF: $378 \times 504$ [S1].
*   **Ray Sampling:**
    *   DTU/Replica: 1024 rays per step [S1].
    *   LLFF: 2048 rays per step [S1].
*   **Embedding Dimension:**
    *   Defined by `opt.arch.embedding_dim` [S2, unofficial]. Not given in the sources.

### Edge Cases and Pitfalls

*   **Background Artifacts:** In LLFF joint optimization, applying multi-view correspondence and depth consistency losses from the start can cause the background to appear in front of the foreground. It is recommended to use only photometric loss for the first 1,000 iterations [S1].
*   **Hierarchical Sampling:** For LLFF, hierarchical sampling provides no significant benefit over a single MLP and increases training time. A single MLP is preferred for efficiency [S1].
*   **Precision:** The Gram-Schmidt process for rotation reconstruction requires careful normalization to avoid numerical instability, especially when $a_1$ or the residual vector is near zero [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class LocalSE3WarpRefinement:
    def __init__(self, num_cameras, embedding_dim, warp_mlp):
        self.num_cameras = num_cameras
        self.warp_embedding = nn.Embedding(num_cameras, embedding_dim)
        self.warp_mlp = warp_mlp
        # Initialize embeddings [G]
        nn.init.zeros_(self.warp_embedding.weight)

    def refine_pose(self, initial_pose, camera_indices, ray_indices, intr_inv, batch_size):
        # 1. Gather 3D camera coordinates for sampled rays [S2, unofficial]
        # Source does not provide implementation for gathering 3D coordinates from ray indices [S2, unofficial].
        camera_cords_grid_3D = gather_camera_cords_grid_3D(
            initial_pose, intr_inv, ray_indices, batch_size
        )
        
        # 2. Extract 2D coordinates [S2, unofficial]
        camera_cords_grid_2D = camera_cords_grid_3D[..., :2]
        
        # 3. Retrieve per-camera embeddings [S2, unofficial]
        embeddings = self.warp_embedding(camera_indices)
        num_rays = camera_cords_grid_2D.shape[1]
        embeddings_expanded = embeddings.unsqueeze(1).expand(-1, num_rays, -1)
        
        # 4. Concatenate 2D coords and embeddings [S2, unofficial]
        input_tensor = torch.cat((camera_cords_grid_2D, embeddings_expanded), dim=-1)
        
        # 5. Pass through Warp MLP [S2, unofficial]
        local_se3_refine = self.warp_mlp(input_tensor)
        
        # 6. Convert SE(3) Lie algebra to SE(3) matrix [S2, unofficial]
        # Source does not provide implementation for SE(3) exponential map [G].
        local_pose_refine = se3_to_SE3(local_se3_refine)
        
        # 7. Compose local refinement with base pose [S2, unofficial]
        base_poses = initial_pose[camera_indices]
        base_poses_expanded = base_poses.unsqueeze(1).expand(-1, num_rays, -1, -1)
        local_pose = torch.matmul(local_pose_refine, base_poses_expanded)
        
        # 8. Aggregate poses (e.g., by averaging or selecting a representative ray)
        refined_pose = local_pose.mean(dim=1)
        
        return refined_pose

# Source does not provide implementation for gathering 3D coordinates from ray indices [S2, unofficial].
def gather_camera_cords_grid_3D(initial_pose, intr_inv, ray_indices, batch_size):
    pass

# Source does not provide implementation for SE(3) exponential map [G].
def se3_to_SE3(lie_algebra):
    pass
```

## Subtopics

### Geometric Constraints for Inliers
This variant, associated with Robust SG-NeRF [S3], enhances the pose refinement process by distinguishing between inlier and outlier poses based on confidence estimation. For inlier poses, it applies additional geometric constraints to improve consistency:
*   **Re-projection Loss:** A loss term based on the current Signed Distance Function (SDF) and pose estimations is applied to inliers to strengthen constraints between matching image pairs [S3].
*   **Intersection-of-Union (IoU) Loss:** This loss is applied to further strengthen constraints between matching image pairs for inliers [S3].
*   **Outlier Handling:** For outlier poses, a Monte Carlo re-localization method is used to find better initializations, and a scene graph updating strategy is employed to eliminate incorrectly matched pairs [S3]. This approach avoids the instability of optimizing all poses equally and improves reconstruction quality on datasets like SG-NeRF and DTU [S3].

### Joint Camera Pose and Generative Model Learning
This variant jointly learns camera pose distribution and generative model [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** The addition of multi-view correspondence loss and depth consistency objectives adds a factor of approximately 1.5 to the optimization time, regardless of the number of views [S1]. The PDC-Net for predicting matches runs at 10fps on $300 \times 400$ images [S1].
*   **Memory Complexity:** The method introduces memory overhead for storing per-camera embeddings and the Warp MLP. The embedding dimension is a hyperparameter [S2, unofficial].
*   **Performance:** The method has been validated on DTU, Replica, and LLFF datasets [S1]. It effectively corrects outlier poses and improves reconstruction quality [S3].
*   **Strengths:**
    *   Joint optimization of poses and radiance fields [S1].
    *   Learnable per-camera embeddings allow for view-specific corrections [S2, unofficial].
*   **Limitations:**
    *   Requires careful tuning of learning rates for network weights vs. poses [S1].
    *   May suffer from background artifacts if multi-view losses are applied too early in LLFF scenes [S1].
    *   Relies on the quality of initial poses; poor initializations may require robust outlier handling [S3].
*   **When to Use:** Not given in the sources.
*   **When Not to Use:** Not given in the sources.

## Background
The method originates from research on optimizing Neural Radiance Fields (NeRF) from sparse and noisy poses. Key contributions include:
*   **SPARF (2023):** Introduced a method for optimizing NeRF from sparse and noisy poses using a Gram-Schmidt-like process for rotation reconstruction and specific learning rate schedules for DTU, Replica, and LLFF datasets [S1].
*   **Joint-TensoRF (Unofficial):** Proposed an embedding-based warp MLP approach for local SE(3) pose correction, leveraging per-camera embeddings and 2D coordinate conditioning [S2, unofficial].
*   **Robust SG-NeRF (2024):** Enhanced the pose refinement process with geometric constraints for inliers and Monte Carlo re-localization for outliers, improving robustness in scene graph-aided reconstruction [S3].
*   **Joint Camera Pose and Generative Model Learning:** Jointly learns camera pose distribution and generative model [S4, unofficial].

## Sources
[S1] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
[S2] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
[S3] Robust SG-NeRF: Robust Scene Graph Aided Neural Surface Reconstruction, https://arxiv.org/abs/2411.13620v1, fetched 2026-10-26
[S4] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
