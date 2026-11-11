# Learnable Pose Refinement via Local SE(3) Warp

Also called: Embedding-Based Warp MLP, Local SE(3) Pose Correction

## Overview
This method refines noisy initial camera poses by learning a per-camera correction in the local tangent space of the SE(3) group, conditioned on 2D image coordinates. It addresses the problem of pose drift in sparse-view or noisy-initialization scenarios (e.g., COLMAP outputs) by jointly optimizing the neural radiance field and camera extrinsics. Unlike global pose optimization methods that treat poses as independent vectors, this approach models pose corrections as a continuous function of pixel location, allowing for spatially coherent refinement [S1][S2, unofficial]. It sets itself apart from standard BARF-style coarse-to-fine optimization by using a dedicated MLP to predict local SE(3) warps rather than relying solely on frequency-based positional encoding schedules [S1][S2, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
1.  Initial noisy camera poses $P_{w2c} \in SE(3)$ for $N$ cameras.
2.  Inverse intrinsic matrices $K^{-1}$.
3.  Image dimensions (width $W$, height $H$).
4.  Per-camera learnable embeddings.

**Outputs:**
1.  Refined camera poses $P'_{w2c}$.
2.  Gradients for pose refinement and network weights.

### Data Structures

**Warp Embedding:**
A learnable lookup table storing a vector for each camera.
*   `warp_embedding`: `torch.nn.Embedding(num_cameras, embedding_dim)`
*   Layout: A tensor of shape `(num_cameras, embedding_dim)`.
*   `embedding_dim`: Integer dimensionality of the pose correction embedding [S2, unofficial].

**Warp MLP:**
A neural network mapping 2D coordinates and embeddings to SE(3) Lie algebra vectors.
*   `warp_mlp`: A multi-layer perceptron (MLP).
*   Input dimension: $2 + \text{embedding\_dim}$ (2D coordinates + embedding).
*   Output dimension: 6 (SE(3) Lie algebra vector $\xi \in \mathbb{R}^6$).
*   Architecture: Specific hidden layers are not detailed in the sources, but it maps the concatenated input to a 6D vector [S2, unofficial].

### Algorithm Steps

1.  **Initialize Poses:** Start with initial rotation $\hat{r}_0$ and translation $t_0$ derived from noisy estimates (e.g., COLMAP) [S1].
2.  **Gather Camera Coordinates:** For a batch of rays, compute the 3D camera coordinates grid.
    *   Generate 2D pixel coordinates $(u, v)$.
    *   Unproject to 3D using inverse intrinsics $K^{-1}$ and the current pose to get `camera_cords_grid_3D` [S2, unofficial].
    *   Extract 2D coordinates: `camera_cords_grid_2D = camera_cords_grid_3D[..., :2]` [S2, unofficial].
3.  **Retrieve Embeddings:** For each camera $i$ in the batch, retrieve its specific embedding vector $e_i$.
    *   `embedding = warp_embedding.weight[i, None, :]` expanded to match the number of points [S2, unofficial].
4.  **Compute Local SE(3) Correction:**
    *   Concatenate 2D coordinates and embedding: $x_{in} = \text{concat}(\text{camera\_cords\_grid\_2D}, \text{embedding})$ [S2, unofficial].
    *   Pass through the Warp MLP: $\xi = \text{warp\_mlp}(x_{in})$, where $\xi \in \mathbb{R}^6$ is the Lie algebra vector [S2, unofficial].
5.  **Map to SE(3) Matrix:** Convert the Lie algebra vector $\xi$ to an SE(3) matrix $\Delta P$ using the exponential map.
    *   `local_pose_refine = lie.se3_to_SE3(xi)` [S2, unofficial].
6.  **Compose Pose:** Apply the local correction to the base pose.
    *   `local_pose = compose([local_pose_refine, base_pose])` [S2, unofficial].
    *   This results in the refined pose for the current iteration.
7.  **Update Parameters:**
    *   Update rotation and translation estimates iteratively. The rotation is updated using a Gram-Schmidt-like process to ensure orthogonality [S1].
    *   Let $r$ be the 6D rotation parameterization (see below). The update rule is:
        $$ \hat{R}_{w2c} = f(\hat{r}_{w2c} + \Delta r) $$
        $$ \hat{t}_{w2c} = t_{w2c} + \Delta t $$
        where $\hat{r}_{w2c}$ and $t_{w2c}$ are the current estimates, and $\Delta r, \Delta t$ are the gradients/updates from the optimizer [S1].

### Rotation Parameterization (6D to Matrix)
The method uses a 6D rotation representation $r = [a_1^T, a_2^T]^T$ where $a_1, a_2 \in \mathbb{R}^3$. The rotation matrix $R$ is recovered via a function $f$ using a Gram-Schmidt-like orthogonalization [S1]:

1.  $b_1 = N(a_1)$
2.  $b_2 = N(a_2 - (b_1 \cdot a_2)b_1)$
3.  $b_3 = b_1 \times b_2$
4.  $R = [b_1 \mid b_2 \mid b_3]$

Where $N(\cdot)$ denotes the L2 norm [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class LocalSE3WarpRefinement:
    def __init__(self, num_cameras, embedding_dim, mlp_hidden_dims=[256, 256, 256]):
        """
        Initialize the learnable pose refinement module.
        
        Args:
            num_cameras: Number of cameras in the scene.
            embedding_dim: Dimensionality of the per-camera embedding.
            mlp_hidden_dims: List of hidden layer sizes for the warp MLP.
        """
        self.num_cameras = num_cameras
        self.embedding_dim = embedding_dim
        
        # Data Structures
        # warp_embedding: Stores learnable per-camera pose correction vectors
        self.warp_embedding = nn.Embedding(num_cameras, embedding_dim)
        
        # warp_mlp: Maps concatenated 2D coords and embeddings to 6D SE(3) Lie algebra
        input_dim = 2 + embedding_dim
        layers = []
        for hidden_dim in mlp_hidden_dims:
            layers.append(nn.Linear(input_dim, hidden_dim))
            layers.append(nn.ReLU())
            input_dim = hidden_dim
        layers.append(nn.Linear(input_dim, 6)) # Output 6D Lie algebra vector
        self.warp_mlp = nn.Sequential(*layers)

    def compute_rotation_matrix(self, r):
        """
        Convert 6D rotation vector r = [a1, a2] to 3x3 rotation matrix R.
        Uses Gram-Schmidt-like process [S1].
        
        Args:
            r: Tensor of shape (..., 6)
            
        Returns:
            R: Tensor of shape (..., 3, 3)
        """
        a1 = r[..., :3]
        a2 = r[..., 3:6]
        
        # b1 = N(a1)
        b1 = F.normalize(a1, dim=-1)
        
        # b2 = N(a2 - (b1 . a2) * b1)
        dot_product = torch.sum(b1 * a2, dim=-1, keepdim=True)
        b2 = a2 - dot_product * b1
        b2 = F.normalize(b2, dim=-1)
        
        # b3 = b1 x b2
        b3 = torch.cross(b1, b2, dim=-1)
        
        # R = [b1 b2 b3]
        R = torch.stack([b1, b2, b3], dim=-1)
        return R

    def se3_to_matrix(self, xi):
        """
        Convert SE(3) Lie algebra vector xi (6D) to 4x4 SE(3) matrix.
        xi = [omega, v] where omega is rotation (3D) and v is translation (3D).
        Uses Rodrigues' formula for rotation part [G].
        
        Args:
            xi: Tensor of shape (..., 6)
            
        Returns:
            P: Tensor of shape (..., 4, 4)
        """
        omega = xi[..., :3]
        v = xi[..., 3:6]
        
        # Compute rotation matrix from axis-angle (omega)
        # theta = ||omega||
        theta = torch.norm(omega, dim=-1, keepdim=True)
        # Handle small theta to avoid division by zero
        theta = torch.clamp(theta, min=1e-8)
        
        # Axis of rotation
        k = omega / theta
        
        # Rodrigues' formula
        # R = I + sin(theta)*K + (1-cos(theta))*K^2
        # K is the skew-symmetric matrix of k
        kx = torch.stack([
            torch.zeros_like(k), -k[..., 2], k[..., 1],
            k[..., 2], torch.zeros_like(k), -k[..., 0],
            -k[..., 1], k[..., 0], torch.zeros_like(k)
        ], dim=-1).reshape(*k.shape[:-1], 3, 3)
        
        sin_theta = torch.sin(theta).unsqueeze(-1).unsqueeze(-1)
        cos_theta = torch.cos(theta).unsqueeze(-1).unsqueeze(-1)
        I = torch.eye(3).to(xi.device).unsqueeze(0).expand(*xi.shape[:-1], 3, 3)
        
        R = I + sin_theta * kx + (1 - cos_theta) * torch.matmul(kx, kx)
        
        # Construct 4x4 SE(3) matrix
        # [ R  v ]
        # [ 0  1 ]
        P = torch.zeros((*xi.shape[:-1], 4, 4), device=xi.device)
        P[..., :3, :3] = R
        P[..., :3, 3] = v
        P[..., 3, 3] = 1.0
        
        return P

    def forward(self, base_poses, intr_inv, pixel_coords, camera_indices):
        """
        Compute refined poses for a batch of rays.
        
        Args:
            base_poses: Tensor of shape (num_cameras, 4, 4) or (num_cameras, 3, 4).
                        Initial noisy poses.
            intr_inv: Inverse intrinsic matrix K^-1.
            pixel_coords: Tensor of shape (batch_size, 2) containing (u, v) coordinates.
            camera_indices: Tensor of shape (batch_size,) containing camera IDs for each ray.
            
        Returns:
            refined_poses: Tensor of shape (batch_size, 4, 4) representing the refined 
                           transformation for each ray's camera.
        """
        batch_size = pixel_coords.shape[0]
        
        # 1. Retrieve per-camera embeddings
        # embedding shape: (batch_size, embedding_dim)
        embeddings = self.warp_embedding(camera_indices)
        
        # 2. Concatenate 2D coords and embeddings
        # pixel_coords shape: (batch_size, 2)
        input_tensor = torch.cat([pixel_coords, embeddings], dim=-1)
        
        # 3. Pass through Warp MLP to get 6D Lie algebra vector
        # local_se3_refine shape: (batch_size, 6)
        local_se3_refine = self.warp_mlp(input_tensor)
        
        # 4. Convert SE(3) Lie algebra to 4x4 matrix
        # local_pose_refine shape: (batch_size, 4, 4)
        local_pose_refine = self.se3_to_matrix(local_se3_refine)
        
        # 5. Compose local refinement with base pose
        # Get base pose for each ray's camera
        # base_poses shape: (num_cameras, 4, 4)
        # base_pose_for_ray shape: (batch_size, 4, 4)
        base_pose_for_ray = base_poses[camera_indices]
        
        # Compose: local_pose = local_pose_refine @ base_pose_for_ray
        # Note: The source says compose([local_pose_refine, pose]). 
        # Typically, if local_pose_refine is a correction in camera space, 
        # it might be pre-multiplied. Assuming standard composition P_new = P_corr * P_old
        refined_poses = torch.bmm(local_pose_refine, base_pose_for_ray)
        
        return refined_poses

    def update_rotation(self, current_r, delta_r):
        """
        Update rotation parameters using the 6D representation and Gram-Schmidt [S1].
        
        Args:
            current_r: Current 6D rotation vector (..., 6)
            delta_r: Gradient/Update for rotation (..., 6)
            
        Returns:
            new_R: Updated 3x3 rotation matrix
        """
        new_r = current_r + delta_r
        new_R = self.compute_rotation_matrix(new_r)
        return new_R
```

### Parameters and Typical Values

*   **Learning Rates (Adam Optimizer):**
    *   Network weights: Initial $5 \times 10^{-4}$, decaying exponentially to $1 \times 10^{-4}$ [S1].
    *   Camera poses: Initial $1 \times 10^{-3}$, decaying to $1 \times 10^{-4}$ [S1].
    *   LLFF specific: Network LR $1 \times 10^{-3}$ to $1 \times 10^{-4}$; Pose LR $3 \times 10^{-3}$ to $1 \times 10^{-5}$ [S1].
*   **Image Resolution:**
    *   DTU/Replica: $300 \times 400$ [S1].
    *   LLFF: $378 \times 504$ [S1].
*   **Ray Sampling:**
    *   DTU/Replica: 1024 random pixel rays per step [S1].
    *   LLFF: 2048 random pixel rays per step [S1].
*   **Embedding Dimension:** Not explicitly fixed in sources, defined as `embedding_dim` [S2, unofficial].

### Edge Cases and Pitfalls

1.  **Background Artifacts:** In LLFF joint pose-NeRF optimization, applying multi-view correspondence loss and depth consistency loss from the beginning can cause the background to appear in front of the foreground. The method recommends using only photometric loss for the first 1,000 iterations [S1].
2.  **Rotation Orthogonality:** The 6D rotation representation requires Gram-Schmidt orthogonalization at every update step to ensure the resulting matrix is a valid rotation matrix [S1].
3.  **Precision:** The exponential decay of learning rates is critical for convergence. Stagnation may occur if the decay is too slow or too fast [S1].
4.  **Hierarchical Sampling:** For LLFF, hierarchical sampling was found unnecessary; a single MLP yields the same results with faster training [S1].

## Subtopics

### BARF (Blending Artificial Reality Features)
A baseline method that uses frequency-based positional encoding schedules to optimize poses and NeRF jointly. It does not use a separate warp MLP but relies on the NeRF's capacity to learn low-frequency pose corrections first [S1].

### SCNeRF
A baseline method using official code for comparison. It typically involves separate pose optimization steps or different regularization strategies [S1].

## Operating characteristics

*   **Time Complexity:** The addition of multi-view correspondence loss and depth consistency objectives adds a factor of approximately 1.5 to the optimization time, regardless of the number of views [S1]. The PDC-Net (used for correspondence) runs at 10 fps on $300 \times 400$ images [S1].
*   **Memory Complexity:** Depends on the size of the `warp_embedding` and `warp_mlp`. The embedding size is $O(N \cdot D)$ where $N$ is cameras and $D$ is embedding dim. The MLP size is constant relative to scene size.
*   **Strengths:**
    *   Handles noisy initial poses effectively.
    *   Spatially coherent refinement via the 2D coordinate conditioning.
    *   Reduces training time on LLFF by avoiding hierarchical sampling [S1].
*   **Limitations:**
    *   Requires careful tuning of learning rate schedules.
    *   May require delayed addition of consistency losses to avoid artifacts [S1].
    *   Performance depends on the quality of the initial COLMAP reconstruction [S1].
*   **When to Use:** Sparse-view reconstruction, scenes with noisy initial poses, or when global optimization methods fail to converge.
*   **When Not to Use:** Dense-view scenarios with high-quality initial poses where the overhead of learning local warps is unnecessary.

## Background

The method originates from the SPARF paper (Sparse and Noisy Poses), which addresses the challenge of optimizing NeRFs from sparse and noisy camera poses [S1]. It builds on the idea of blending artificial reality features (BARF) but introduces a learnable local warp to refine poses more effectively [S1]. The implementation details regarding the warp MLP and embeddings are also found in unofficial repositories like Joint-TensoRF [S2, unofficial]. The 6D rotation representation is based on work by Zhou et al. [63] in the SPARF paper [S1].

## Sources

[S1] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
[S2] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
