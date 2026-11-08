# Camera Pose Optimization

Also called: Pose Optimization
## Overview
Camera Pose Optimization computes gradients of a reconstruction loss with respect to the rotation and translation components of a camera view matrix [S1, unofficial]. It refines an initial camera pose toward a ground truth pose by minimizing gradients derived from 2D depth and binary silhouette observations [Parent]. Unlike methods that optimize scene parameters (e.g., Gaussian centers or NeRF weights), this method treats the camera extrinsics as differentiable variables to correct viewpoint errors [S1, unofficial].
## How it works
### Inputs and outputs
- **Inputs**:
  - Reconstruction loss $\mathcal{L}$ [S1, unofficial].
  - Current camera pose defined by rotation matrix $\mathbf{R}$ and translation vector $\mathbf{t}$ [S1, unofficial].
  - Mean positions $\mu_n$ of the $n$-th Gaussian primitives in the scene [S1, unofficial].
  - Gradients of the loss with respect to the transformed Gaussian centers $\tilde{\mu}_n$ [S1, unofficial].
- **Outputs**:
  - Gradient $\frac{\delta \mathcal{L}}{\delta \mathbf{t}}$ with respect to translation [S1, unofficial].
  - Gradient $\frac{\delta \mathcal{L}}{\delta \mathbf{R}}$ with respect to rotation [S1, unofficial].

### Data structures
- **$\mu_n$**: 3D vector representing the mean of the $n$-th Gaussian [S1, unofficial].
- **$\tilde{\mu}_n$**: Transformed 3D coordinate of the $n$-th Gaussian in camera space [S1, unofficial].
- **$\mathbf{R}$**: $3 \times 3$ rotation matrix [S1, unofficial].
- **$\mathbf{t}$**: $3 \times 1$ translation vector [S1, unofficial].

### Steps
1. **Compute Translation Gradient**:
   Calculate the gradient of the loss with respect to the translation component by summing the gradients with respect to the transformed Gaussian centers [S1, unofficial]:
   $$
   \frac{\delta \mathcal{L}}{\delta \mathbf{t}} = - \sum_n \frac{\delta \mathcal{L}}{\delta \tilde{\mu}_n}
   $$
   [S1, unofficial]

2. **Compute Rotation Gradient**:
   Calculate the gradient of the loss with respect to the rotation component using the translation gradient and the relative positions of the Gaussians [S1, unofficial]:
   $$
   \frac{\delta \mathcal{L}}{\delta \mathbf{R}} = - \left[ \sum_n \frac{\delta \mathcal{L}}{\delta \tilde{\mu}_n} (\mu_n - \mathbf{t})^\top \right] \mathbf{R}
   $$
   [S1, unofficial]

3. **Update Pose**:
   Use the computed gradients to update $\mathbf{R}$ and $\mathbf{t}$ via an optimizer (e.g., Adam or SGD) [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_pose_gradients(loss_grad_transformed_centers, mu, R, t):
    """
    Computes gradients of the reconstruction loss with respect to camera pose.
    
    Args:
        loss_grad_transformed_centers: Tensor of shape (N, 3), containing 
                                       dL/d(mu_tilde_n) for N Gaussians.
        mu: Tensor of shape (N, 3), containing the mean positions of Gaussians.
        R: Tensor of shape (3, 3), the current rotation matrix.
        t: Tensor of shape (3,), the current translation vector.
        
    Returns:
        grad_t: Tensor of shape (3,), gradient w.r.t translation.
        grad_R: Tensor of shape (3, 3), gradient w.r.t rotation.
    """
    # Step 1: Compute translation gradient
    # dL/dt = - sum_n (dL/d(mu_tilde_n))
    grad_t = -torch.sum(loss_grad_transformed_centers, dim=0)
    
    # Step 2: Compute rotation gradient
    # dL/dR = - [ sum_n (dL/d(mu_tilde_n) * (mu_n - t)^T) ] * R
    # (mu_n - t) is shape (N, 3). loss_grad_transformed_centers is (N, 3).
    # We need the outer product sum.
    
    # Calculate relative positions: (N, 3)
    relative_pos = mu - t.unsqueeze(0)
    
    # Calculate the term inside the bracket: sum_n (grad_center * relative_pos^T)
    # This results in a (3, 3) matrix.
    # grad_center is (N, 3), relative_pos is (N, 3).
    # We want sum_n (grad_center[n] * relative_pos[n].T)
    # This is equivalent to grad_center.T @ relative_pos
    bracket_term = loss_grad_transformed_centers.T @ relative_pos
    
    # Multiply by R
    grad_R = - (bracket_term @ R)
    
    return grad_t, grad_R
```

### Parameters
- **$\lambda_{\text{noise}}$**: Hyperparameter controlling the noise term applied to Gaussian centers [S1, unofficial].
- **$\lambda_{\text{lr}}$**: Learning rate hyperparameter [S1, unofficial].
- **$\epsilon$**: Noise term applied to the center locations $\mu$ [S1, unofficial].

### Edge cases and pitfalls
- **Singularity**: The rotation gradient formula assumes $\mathbf{R}$ is a valid rotation matrix. If $\mathbf{R}$ drifts from orthogonality during optimization, the gradient may become inaccurate. Re-orthogonalization (e.g., via SVD) may be required after updates [G].
- **Zero Division**: Not applicable in the gradient formulas themselves, but depth rendering steps (used in loss calculation) may involve division by accumulated alpha or depth, requiring epsilon guards [S1, unofficial].

## Subtopics

### Learnable Pose Refinement via Local SE(3) Warp
A variant that uses a neural network to predict local pose corrections. It gathers 3D camera coordinates, extracts 2D coordinates, retrieves per-camera embeddings, and passes them through a warp MLP to output 6D SE(3) Lie algebra vectors [S2, unofficial]. These are converted to SE(3) matrices and composed with the base pose [S2, unofficial]. It is chosen when global optimization is unstable or when per-camera systematic errors exist [S2, unofficial]. It uses `warp_embedding` (stores learnable per-camera pose correction vectors) and `warp_mlp` (maps concatenated 2D coords and embeddings to 6D SE(3) Lie algebra vectors) [S2, unofficial]. Parameters include `embedding_dim` [S2, unofficial].

## Operating characteristics
- **Time Complexity**: Linear in the number of Gaussians $N$ for gradient computation [S1, unofficial].
- **Memory Complexity**: Linear in $N$ to store gradients and positions [S1, unofficial].
- **Strengths**: Directly optimizes camera extrinsics, useful for structure-from-motion refinement or correcting initial pose estimates [S1, unofficial].
- **Limitations**: Requires differentiable rendering of the scene (e.g., Gaussian Splatting or NeRF) to compute gradients [S1, unofficial]. Sensitive to initialization; poor initial poses may lead to local minima [G].

## Background
Originated in the context of differentiable rendering, specifically Gaussian Splatting, where camera poses are optimized alongside or after scene parameters [S1, unofficial]. The method is detailed in the `gsplat` library documentation [S1, unofficial]. Variants using learnable embeddings for pose refinement appear in joint optimization frameworks like Joint-TensoRF [S2, unofficial].

## Sources
[S1] gsplat: An Open-Source Library for Gaussian Splatting, https://doi.org/10.48550/arXiv.2409.06765, fetched 2026-10-29 (unofficial)
[S2] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
