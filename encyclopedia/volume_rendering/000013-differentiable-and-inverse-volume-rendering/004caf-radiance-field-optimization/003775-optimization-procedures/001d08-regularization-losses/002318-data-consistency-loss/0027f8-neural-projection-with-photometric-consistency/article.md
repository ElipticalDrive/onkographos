# Neural Projection with Photometric Consistency

Also called: Point-to-Image Photometric Constraint

## Overview
Neural Projection with Photometric Consistency is a regularization technique for implicit surface reconstruction that enforces geometric accuracy by projecting sampled points onto the learned implicit surface and verifying their appearance consistency across multiple views [S1, unofficial]. It solves the problem of noisy SDF predictions by using multi-view photometric constraints to supervise the position of the projected surface points, indirectly ensuring the accuracy of the SDF values at the original sample locations [S1, unofficial]. Unlike sibling methods that rely on depth maps or optimal transport, this method operates directly on the implicit field's gradient to map points to the surface, making it particularly effective for handling noisy data where explicit depth estimation is unreliable [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D points $P$ sampled in space; a base Signed Distance Function (SDF) network $f$ that outputs scalar SDF values and gradients; a set of $N$ reference and source images $\{I_r, I_{s_1}, ..., I_{s_N}\}$ with known camera poses; a patch size $s$ for image extraction.
*   **Outputs:** A photometric consistency loss value $\mathcal{L}_{pc}$ used to update the base SDF network.

**Data Structures:**
*   **Point Set $P$:** A list of 3D coordinates $x_P \in \mathbb{R}^3$.
*   **SDF Network $f$:** A neural network taking $x_P$ as input, outputting $f(x_P) \in \mathbb{R}$ and $\nabla f(x_P) \in \mathbb{R}^3$.
*   **Image Patches:** 2D image regions extracted from reference and source views centered at projected coordinates.

**Numbered Steps:**
1.  **Gradient Computation:** For each point $x_P \in P$, compute the gradient of the SDF field:
    $$ g(x_P) = \nabla f(x_P) $$
    The gradient direction $g(x_P)$ represents the direction of the fastest increasing SDF value and points outward from the surface [S1, unofficial].

2.  **Neural Projection:** Project the point $x_P$ to its nearest neighbor $t_P$ on the implicit surface $S$ by moving along the gradient direction by a distance equal to the absolute SDF value $|f(x_P)|$. The projection formula is:
    $$ t_P = x_P - f(x_P) \frac{g(x_P)}{\|g(x_P)\|_2} $$
    This step ensures $t_P$ lies on the zero-level set of the SDF (the implicit surface) [S1, unofficial].

3.  **Patch Extraction:** Identify the reference image $I_r$ corresponding to the depth map that generated the original point $x_P$. Extract a small image patch $s$ centered at the projected coordinate $t_P$ in the reference image $I_r$. Similarly, project $t_P$ into each source image $I_{s_i}$ and extract the corresponding patch $I_{s_i}(s)$ [S1, unofficial].

4.  **Photometric Scoring:** Calculate the Normalized Cross-Correlation (NCC) score between the reference patch $I_r(s)$ and each source patch $I_{s_i}(s)$:
    $$ \text{Score}(I_r(s), I_{s_i}(s)) = \frac{\text{Cov}(I_r(s), I_{s_i}(s))}{\sqrt{\text{Var}(I_r(s)) \text{Var}(I_{s_i}(s))}} $$
    where $\text{Cov}$ is the covariance and $\text{Var}$ is the variance of the pixel intensities in the respective patches [S1, unofficial].

5.  **Loss Aggregation:** Compute the photometric consistency loss $\mathcal{L}_{pc}$ by averaging the inconsistency $(1 - \text{Score})$ over all points in $P$ and the top $m$ source images that yield the best scores:
    $$ \mathcal{L}_{pc} = \frac{1}{m D_P} \sum_{x_P \in P} \sum_{i=1}^m (1 - \text{Score}(I_r(s), I_{s_i}(s))) $$
    where $D_P$ is the number of points in set $P$, and $m$ is the number of source images selected [S1, unofficial].

**Parameters:**
*   $m$: Number of source images used for loss calculation. Set to 4 [S1, unofficial].
*   $s$: Size of the image patch. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Gradient Normalization:** If $\|g(x_P)\|_2$ is near zero, the projection becomes unstable. This typically occurs far from the surface or in flat regions. [G] A small epsilon ($\epsilon$) should be added to the denominator to prevent division by zero: $\|g(x_P)\|_2 + \epsilon$.
*   **Network Application:** The loss is applied to the signals from the **base SDF network**, not the final SDF (which may include a bias network). This is crucial because the projection requires differentiable gradients of the SDF field, and applying the constraint to the final output could interfere with the bias network's ability to refine details [S1, unofficial].
*   **Occlusion:** The method assumes visibility. If a point is occluded in a source view, the photometric score will be low, potentially penalizing valid geometry. The selection of the "top $m$" scores helps mitigate this by ignoring poorly matching views, which often correspond to occlusions or non-Lambertian regions [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def neural_projection_photometric_loss(
    points: torch.Tensor, 
    sdf_net: torch.nn.Module, 
    images: torch.Tensor, 
    camera_poses: torch.Tensor, 
    patch_size: int, 
    m: int = 4
) -> torch.Tensor:
    """
    Computes the Neural Projection with Photometric Consistency loss.
    
    Args:
        points: (B, N, 3) Batch of 3D points.
        sdf_net: Base SDF network returning (sdf_values, gradients).
        images: (B, N_cam, H, W, 3) RGB images for all cameras.
        camera_poses: (B, N_cam, 4, 4) Camera pose matrices.
        patch_size: Size of the square patch for NCC.
        m: Number of best source images to use.
        
    Returns:
        loss: Scalar photometric consistency loss.
    """
    # 1. Compute SDF and Gradient
    # sdf_net returns (B, N, 1) and (B, N, 3)
    sdf_values, gradients = sdf_net(points)
    
    # 2. Project points to surface
    # Normalize gradient
    grad_norm = torch.norm(gradients, dim=-1, keepdim=True) + 1e-8
    normalized_grad = gradients / grad_norm
    
    # t_P = x_P - f(x_P) * g(x_P) / ||g(x_P)||
    # Note: f(x_P) is signed distance. If point is outside, f > 0, 
    # gradient points out, so we subtract to go in.
    projected_points = points - sdf_values * normalized_grad
    
    # 3. Identify Reference and Source Images
    # Assume points were sampled from depth maps of specific cameras.
    # For this implementation, we assume the first camera is reference 
    # and others are sources for simplicity, or use a mapping.
    # In practice, 'reference_idx' would be provided or derived from sampling strategy.
    # Here we assume points are associated with reference camera 0.
    ref_idx = 0
    ref_image = images[:, ref_idx]  # (B, H, W, 3)
    
    # Source images are all others
    source_indices = [i for i in range(images.shape[1]) if i != ref_idx]
    
    # 4. Extract Patches and Compute NCC
    # Project 3D points to 2D image coordinates for all views
    # This requires inverse projection using camera poses.
    # For brevity, assume a helper function 'project_3d_to_2d' exists.
    # ref_coords: (B, N, 2)
    # source_coords: (B, N, N_sources, 2)
    
    ref_coords = project_3d_to_2d(projected_points, camera_poses[:, ref_idx])
    
    # Extract reference patches
    # grid_sample expects normalized coordinates [-1, 1]
    ref_patches = extract_patches(ref_image, ref_coords, patch_size) # (B, N, C, H_p, W_p)
    
    # Initialize loss accumulator
    total_loss = 0.0
    count = 0
    
    # 5. Compute scores for source images
    scores = []
    for src_idx in source_indices:
        src_image = images[:, src_idx]
        src_coords = project_3d_to_2d(projected_points, camera_poses[:, src_idx])
        src_patches = extract_patches(src_image, src_coords, patch_size)
        
        # Compute NCC between ref_patches and src_patches
        # NCC = Cov / sqrt(Var_ref * Var_src)
        # Flatten patches for computation
        ref_flat = ref_patches.view(ref_patches.shape[0], ref_patches.shape[1], -1)
        src_flat = src_patches.view(src_patches.shape[0], src_patches.shape[1], -1)
        
        # Mean and Variance
        ref_mean = ref_flat.mean(dim=-1, keepdim=True)
        src_mean = src_flat.mean(dim=-1, keepdim=True)
        
        ref_centered = ref_flat - ref_mean
        src_centered = src_flat - src_mean
        
        cov = (ref_centered * src_centered).mean(dim=-1)
        var_ref = (ref_centered ** 2).mean(dim=-1)
        var_src = (src_centered ** 2).mean(dim=-1)
        
        ncc = cov / torch.sqrt(var_ref * var_src + 1e-8)
        scores.append(ncc)
        
    # Stack scores: (B, N, N_sources)
    scores_tensor = torch.stack(scores, dim=-1)
    
    # Select top m scores
    # We want the highest NCC (closest to 1), so we take the largest values.
    # However, the loss is (1 - Score), so we want the highest scores to minimize loss.
    # Sort scores in descending order and take top m
    top_scores, _ = torch.topk(scores_tensor, m, dim=-1) # (B, N, m)
    
    # Compute loss: 1 - Score
    # Average over m and N
    # Note: The formula sums over m. We average over m to keep scale consistent.
    # L_pc = 1/(m*D_P) * sum(1 - Score)
    # Here we compute mean over m, then mean over N.
    
    inconsistency = 1.0 - top_scores # (B, N, m)
    loss_per_point = inconsistency.mean(dim=-1) # (B, N)
    loss = loss_per_point.mean() # Scalar
    
    return loss

def extract_patches(image, coords, patch_size):
    """
    Extracts patches from image at given 2D coordinates.
    image: (B, H, W, 3)
    coords: (B, N, 2) in pixel coordinates
    """
    # Convert pixel coordinates to normalized [-1, 1] for grid_sample
    H, W = image.shape[1], image.shape[2]
    norm_coords = 2.0 * coords / torch.tensor([W, H], device=coords.device) - 1.0
    norm_coords = norm_coords.unsqueeze(2) # (B, N, 1, 2)
    
    # grid_sample expects (B, C, H, W) or (B, H, W, C) depending on mode
    # We use 'bilinear' and 'zeros' padding
    patches = F.grid_sample(
        image.permute(0, 3, 1, 2), # (B, 3, H, W)
        norm_coords,
        mode='bilinear',
        padding_mode='zeros',
        align_corners=False
    )
    # Output: (B, 3, N, patch_size, patch_size)
    # Transpose to (B, N, 3, patch_size, patch_size)
    patches = patches.permute(0, 2, 1, 3, 4)
    return patches

def project_3d_to_2d(points_3d, camera_pose):
    """
    Projects 3D points to 2D image coordinates using camera pose.
    points_3d: (B, N, 3)
    camera_pose: (B, 4, 4)
    """
    # Homogeneous coordinates
    points_h = torch.cat([points_3d, torch.ones_like(points_3d[..., :1])], dim=-1) # (B, N, 4)
    
    # Inverse camera pose to transform points to camera space
    # Assuming camera_pose is World-to-Camera? Or Camera-to-World?
    # Standard: P_cam = K * [R|t] * P_world.
    # If camera_pose is Camera-to-World, we need inverse.
    # Let's assume camera_pose is World-to-Camera for simplicity in this helper.
    # If it's Camera-to-World, use torch.inverse(camera_pose).
    
    # Transform points
    points_cam = torch.matmul(points_h, camera_pose.transpose(-1, -2)) # (B, N, 4)
    
    # Extract x, y, z
    x = points_cam[..., 0]
    y = points_cam[..., 1]
    z = points_cam[..., 2]
    
    # Perspective division (assuming K is identity for simplicity, or included in pose)
    # In practice, K is applied. Here we assume normalized coordinates or identity K.
    # u = x / z, v = y / z
    u = x / (z + 1e-8)
    v = y / (z + 1e-8)
    
    # Convert to pixel coordinates (assuming K is identity and image size is known)
    # This is a simplified projection. Real implementation needs K.
    # For this pseudocode, we return normalized coordinates for grid_sample
    # grid_sample expects [-1, 1].
    # u_norm = 2 * u / W - 1
    # v_norm = 2 * v / H - 1
    # We return u, v as is, assuming the caller handles K and normalization.
    return torch.stack([u, v], dim=-1)
```

## Subtopics

### Learned Photometric Consistency
A data-driven variant that casts photo-consistency as a classification task to handle noisy real images and non-Lambertian surfaces [S2]. It uses a shared-weight image encoder to extract features from $N$ multi-view RGB images, projects them to multi-scale local features for each 3D point, and combines them via an attention-based encoder into a single multi-view aware feature. An MLP then outputs a photo-consistency score between 0 and 1 [S2]. Training data is generated by reconstructing surfaces from ground truth point clouds using the Screened Poisson algorithm, rendering depth maps, and sampling positive (on-surface) and negative (off-surface) points with balanced sampling [S2]. To ensure invariance to the number of cameras, camera groups are formed using a soft nearest neighbor approach: a random first camera is selected, its $K'$ nearest neighbors are computed, and $K-1$ cameras are randomly chosen from them, where $K \in [4, 10]$, $K' = \min(2K, 15)$, and total cameras $N \in \{49, 64\}$ [S2].

## Operating characteristics
*   **Time Complexity:** The method requires computing gradients for all sampled points, which is $O(N)$ where $N$ is the number of points. The photometric scoring involves patch extraction and NCC computation for $m$ source images per point, adding a factor of $m$. Thus, the loss computation is roughly $O(N \cdot m)$.
*   **Memory Complexity:** Requires storing the SDF network parameters, the gradients for all points, and the image patches for all points and source views. This scales linearly with $N$ and $m$.
*   **Strengths:** It provides strong geometric constraints directly on the implicit surface, improving accuracy on noisy data where explicit depth maps are unreliable [S1, unofficial]. It is robust to noise because it relies on the consistency of the implicit field's gradient rather than discrete depth values [S1, unofficial].
*   **Limitations:** It assumes Lambertian surfaces for the standard NCC score, which may fail on highly reflective or non-Lambertian surfaces unless the Learned variant is used [S2]. The projection step requires differentiable SDF gradients, which can be unstable if the network is not well-trained or if points are far from the surface [S1, unofficial].
*   **When to Use:** Use when reconstructing surfaces from noisy multi-view data where depth estimation is unreliable, and the scene has sufficient texture for photometric matching [S1, unofficial].
*   **When Not to Use:** Avoid for scenes with large non-Lambertian regions if using the standard NCC variant, or when computational resources are extremely limited due to the per-point gradient and patch extraction costs [S1, unofficial][S2].

## Background
The method was introduced in "Point-NeuS: Point-Guided Neural Implicit Surface Reconstruction by Volume Rendering" [S1, unofficial]. It builds upon the idea of using multi-view stereo (MVS) photometric consistency to regularize neural implicit surfaces. The core insight is that points projected onto the implicit surface should exhibit consistent appearance across multiple views, providing a strong geometric prior [S1, unofficial]. The learned variant was proposed in "Multi-View Reconstruction Using Signed Ray Distance Functions (SRDF)" to address limitations of hand-crafted photometric priors on real-world noisy data [S2].

## Sources
[S1] Point-NeuS: Point-Guided Neural Implicit Surface Reconstruction by Volume Rendering, https://doi.org/10.48550/arXiv.2310.07997, fetched 2026-10-30 (unofficial)
[S2] Multi-View Reconstruction Using Signed Ray Distance Functions (SRDF), https://doi.org/10.1109/cvpr52729.2023.01602, fetched 2026-10-26
