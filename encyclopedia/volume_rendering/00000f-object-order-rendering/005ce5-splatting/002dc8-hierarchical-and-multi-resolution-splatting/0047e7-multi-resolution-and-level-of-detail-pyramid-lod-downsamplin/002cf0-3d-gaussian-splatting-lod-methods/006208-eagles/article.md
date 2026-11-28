# EAGLES

Also called: Efficient Accelerated 3D Gaussians with Lightweight EncodingS

## Overview
EAGLES is a method for optimizing 3D Gaussian Splatting (3D-GS) representations by reducing memory footprint, training time, and storage size while maintaining or improving reconstruction quality [S3]. It achieves this through attribute quantization, progressive coarse-to-fine training, and aggressive pruning [S3]. Unlike standard 3D-GS, which stores full-precision attributes for millions of Gaussians, EAGLES compresses color, rotation, opacity, position, and scale into latent vectors decoded by a small MLP, and reduces the Gaussian count via influence-based pruning [S1, S3]. It sets itself apart from sibling LOD methods like H-3DGS by focusing on attribute-level compression and training-time acceleration rather than hierarchical spatial structures [S3].

## How it works
### Inputs and Outputs
**Inputs:**
*   Sparse point cloud from Structure from Motion (SfM) [S3].
*   Training images and camera poses [S3].
*   Hyperparameters: SH degree $f$, latent dimension $l$, decoder output dimension $k$, progressive scaling start/end scales, densification interval, and pruning thresholds [S3].

**Outputs:**
*   Quantized latent vectors $q$ for each Gaussian [S3].
*   Trained MLP decoder $D$ [S3].
*   Rendered novel views [S3].

### Data Structures
Each Gaussian $i$ stores:
1.  **Latent Vector** $q_i \in \mathbb{Z}^l$: Integer vector of dimension $l$ [S3].
2.  **Decoded Attributes**: The MLP decoder $D: \mathbb{Z}^l \rightarrow \mathbb{R}^k$ maps $q_i$ to rotation $r_i$, opacity $o_i$, color SH coefficients $c_{SH, i}$, position $p_i$, and scale $s_i$ [S3]. The dimension $k$ depends on $f$, where $d = 3f^2$ coefficients are required for color [S3].

### Steps

1.  **Initialization:** Initialize Gaussians from the SfM point cloud with position $p$, scale $s$, and continuous latent approximations $\hat{q}$ [S3].
2.  **Attribute Quantization (Training):**
    *   Maintain continuous approximations $\hat{q}$ for integer latents $q$ during training [S3].
    *   Compute decoded attributes using the Straight-Through Estimator (STE):
        $$ \boldsymbol{a} = D(\text{STE}(\hat{q})) $$
        where $\text{STE}(\hat{q})$ rounds $\hat{q}$ to the nearest integer for the forward pass but passes gradients through $\hat{q}$ in the backward pass [S3].
    *   Reconstruct the 3D Gaussian covariance matrix $\Sigma$ from decoded rotation $r$ and scale $s$:
        $$ \Sigma = RSS^TR^T $$
        where $S$ is the diagonal scale matrix from $s$, and $R$ is the rotation matrix from quaternion $r$ [S3].
    *   Project $\Sigma$ to 2D for the current camera view:
        $$ \Sigma' = JP\Sigma P^TJ^T $$
        where $P$ is the world-to-camera projective transform and $J$ is the Jacobian of the affine approximation of the projective transform [S3].
    *   Compute opacity $\alpha$ and color $c$ for rasterization. Opacity $\alpha$ is derived from the 2D Gaussian value at the pixel multiplied by the decoded scalar opacity $o$ [S3]. Color $c$ is computed from the decoded SH coefficients [S3].
    *   Render the image and compute loss:
        $$ \mathcal{L} = (1-\lambda)\mathcal{L}_1 + \lambda\mathcal{L}_{\text{SSIM}} $$
        with $\lambda = 0.2$ [S3].
    *   Update $\hat{q}$, $p$, and $s$ via Stochastic Gradient Descent [S3].
3.  **Progressive Coarse-to-Fine Training:**
    *   Start rendering at a reduced resolution scale (e.g., 0.3 of original dimensions) [S3].
    *   Gradually increase the rendering resolution to 1.0 (full scale) over a cosine schedule spanning 70% of the total training iterations [S3].
    *   Alternatively, use Gaussian filtering (e.g., 5x5, 7x7, or 15x15 kernel) or mean filtering during this phase to stabilize optimization [S3]. Downsampling provides the largest training time gains [S3].
4.  **Densification and Pruning:**
    *   **Warmup:** Do not prune or densify for the initial phase [S3].
    *   **Pruning:** Every 100 iterations, remove Gaussians with opacity $\alpha < \alpha_{\text{thresh}}$ [S3].
    *   **Densification:** Every 100 iterations, split large Gaussians (covariance larger than geometry) and clone small Gaussians. Only perform this on Gaussians with positional gradients above a threshold $\tau_{\text{thresh}}$ [S3].
    *   **Interval Adjustment:** Increase the densification interval to 175 iterations to reduce the final Gaussian count without quality loss [S3].
    *   **Final Pruning:** After densification stabilizes, apply a pruning stage to remove Gaussians with the least influence on the full reconstruction, using a weight composed of transmittance and opacity [S2].
5.  **Post-Training Compression:**
    *   Round $\hat{q}$ to the nearest integer $q$ [S3].
    *   Apply entropy coding to store $q$ efficiently [S3].
    *   Store the MLP decoder $D$ and the quantized latents [S3].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def ste_round(x):
    """Straight-Through Estimator: forward pass returns rounded integer,
    backward pass passes gradients through the continuous approximation."""
    return np.round(x)

def decode_attributes(latent_vector, decoder_mlp):
    """Decode quantized latent vector to rotation, opacity, color SH, position, and scale."""
    return decoder_mlp(latent_vector)

def compute_covariance_3d(scale, rotation_quaternion):
    """Compute 3D covariance matrix Sigma = R S S^T R^T."""
    S = np.diag(scale)
    q = rotation_quaternion / np.linalg.norm(rotation_quaternion)
    R = np.array([
        [1 - 2*q[2]**2 - 2*q[3]**2, 2*q[1]*q[2] - 2*q[3]*q[0], 2*q[1]*q[3] + 2*q[2]*q[0]],
        [2*q[1]*q[2] + 2*q[3]*q[0], 1 - 2*q[1]**2 - 2*q[3]**2, 2*q[2]*q[3] - 2*q[1]*q[0]],
        [2*q[1]*q[3] - 2*q[2]*q[0], 2*q[2]*q[3] + 2*q[1]*q[0], 1 - 2*q[1]**2 - 2*q[2]**2]
    ])
    return R @ S @ S.T @ R.T

def project_covariance_2d(cov_3d, camera_matrix, jacobian):
    """Project 3D covariance to 2D: Sigma' = J P Sigma P^T J^T."""
    P = camera_matrix
    return jacobian @ P @ cov_3d @ P.T @ jacobian.T

def render_gaussians(gaussians, camera, image_width, image_height, scale_factor):
    """Render image using tile-based rasterization."""
    target_w = int(image_width * scale_factor)
    target_h = int(image_height * scale_factor)
    rendered_image = np.zeros((target_h, target_w, 3))
    
    for g in gaussians:
        attrs = decode_attributes(g.latent_q, g.decoder)
        rot, opacity, sh_coeffs, pos, scale = attrs['rotation'], attrs['opacity'], attrs['sh_coeffs'], attrs['position'], attrs['scale']
        
        cov_3d = compute_covariance_3d(scale, rot)
        proj_cov_2d = project_covariance_2d(cov_3d, camera.P, camera.J)
        cov_2d_inv = np.linalg.inv(proj_cov_2d)
        
        center_2d = camera.project(pos)
        # For each pixel in the Gaussian's bounding box:
        # dx, dy = pixel_coords - center_2d
        # alpha = np.exp(-0.5 * (dx**2 * cov_2d_inv[0,0] + 2*dx*dy * cov_2d_inv[0,1] + dy**2 * cov_2d_inv[1,1])) * opacity
        # color = evaluate_sh(sh_coeffs, view_direction)
        # Alpha compositing: C = sum(c_i * alpha_i * product(1 - alpha_j)) [S3]
        pass 
        
    return rendered_image

def train_eagles(gaussians, dataset, decoder_mlp, total_iterations):
    """Main training loop."""
    start_scale = 0.3
    end_scale = 1.0
    progressive_duration = int(0.7 * total_iterations)
    warmup_iterations = 3000 # Standard warmup phase [G]
    prune_interval = 100
    densify_interval = 100
    alpha_thresh = 0.005 # Threshold for opacity pruning [G]
    grad_thresh = 0.0002 # Threshold for positional gradients [G]
    
    for iter in range(total_iterations):
        if iter < progressive_duration:
            progress = iter / progressive_duration
            scale = start_scale + (end_scale - start_scale) * 0.5 * (1 - np.cos(np.pi * progress))
        else:
            scale = 1.0
            
        image, camera = dataset.sample()
        rendered = render_gaussians(gaussians, camera, image.shape[1], image.shape[0], scale)
        
        # Loss: L = (1-0.2)*L1 + 0.2*L_SSIM [S3]
        # loss = compute_loss(rendered, image) # Standard L1+SSIM combination [G]
        
        # Backward pass with STE: gradients flow through scale, position, and continuous latent approx q_hat
        # gradients = compute_gradients(loss, gaussians, decoder_mlp) # Standard SGD step [G]
        # update_parameters(gaussians, decoder_mlp, gradients, lr=1e-4)
        
        if iter > warmup_iterations:
            if iter % prune_interval == 0:
                gaussians = [g for g in gaussians if g.opacity > alpha_thresh]
                
            if iter % densify_interval == 0:
                # Split large Gaussians and clone small ones based on positional gradients > grad_thresh [S3]
                # gaussians = densify_gaussians(gaussians, grad_thresh) # Standard densification logic [G]
                
            if iter > 10000: 
                densify_interval = 175 # Increase interval to reduce Gaussian count [S3]
                
    # Final pruning: remove Gaussians with minimal influence using transmittance and opacity weight [S2]
    # gaussians = prune_influence(gaussians, dataset.test_views) # Influence metric calculation [G]
    
    for g in gaussians:
        g.latent_q = np.round(g.latent_q_hat)
        
    return gaussians, decoder_mlp
```

### Parameters
*   **SH Degree $f$:** Typically 4. Color coefficients ($3f^2$) constitute >80% of attribute dimensions [S3].
*   **Latent Dimension $l$:** Dimension of $q \in \mathbb{Z}^l$ [S3].
*   **Decoder Output Dimension $k$:** Dimension of $D: \mathbb{Z}^l \rightarrow \mathbb{R}^k$ [S3].
*   **Progressive Scaling Start:** 0.3 (30% resolution) [S3].
*   **Progressive Scaling End:** 1.0 (100% resolution) [S3].
*   **Progressive Scaling Duration:** 70% of total iterations [S3].
*   **Densification Interval:** 175 iterations (increased from standard 100) [S3].
*   **Gaussian Filter Size:** 5x5, 7x7, or 15x15 if using Gaussian filtering instead of downsampling [S3].

### Pitfalls
*   **Opacity Quantization:** Quantizing opacity is critical to prevent erroneous Gaussians from saturating quickly, which reduces floaters and visual artifacts [S1, S3].
*   **Densification Interval:** Increasing the interval beyond 175 causes a sharp drop in reconstruction quality [S3].
*   **Quantization Drop:** Quantization causes a small drop in PSNR, but opacity quantization can actually improve visual quality by reducing artifacts [S3].
*   **Full Attribute Quantization:** EAGLES quantizes all attributes, including position and scale, via the latent framework to achieve maximum compression [S1].

## Operating characteristics
*   **Memory Reduction:** EAGLES reduces storage by 10-20x compared to 3D-GS [S3]. For example, on the Deep Blending dataset, storage drops from 676 MB (3D-GS) to 52 MB (EAGLES) [S3]. On the Train scene (Tanks & Temples), it drops from 262 MB to 21 MB [S3].
*   **Training Speed:** Training time is significantly reduced. On Deep Blending, EAGLES takes 21m50s vs 36m2s for 3D-GS [S3]. On the Bicycle scene, GPU memory usage drops from 17.4 GB (3D-GS) to 10 GB (EAGLES) [S3].
*   **Rendering Speed:** Rendering FPS is comparable or slightly lower than 3D-GS. On Deep Blending, EAGLES achieves 130 FPS vs 137 FPS for 3D-GS [S3]. EAGLES-Small achieves 160 FPS [S3].
*   **Quality:** PSNR is on par or slightly better. On Deep Blending, EAGLES achieves 29.86 PSNR vs 29.41 PSNR for 3D-GS [S3]. On MipNeRF360, the drop is minimal (0.22 dB) [S3].
*   **Complexity:** The method adds complexity due to the MLP decoder and quantization logic, but reduces the number of Gaussians, which lowers rasterization cost [S3].

## Background
EAGLES (Efficient Accelerated 3D Gaussians with Lightweight EncodingS) was proposed by Sharath Girish, Kamal Gupta, and Abhinav Shrivastava from the University of Maryland, College Park [S3]. It builds upon the seminal 3D Gaussian Splatting work [S3]. The method addresses the high memory and training costs of 3D-GS by introducing attribute quantization, progressive training, and pruning [S3].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] 3DGS.zip: A survey on 3D Gaussian Splatting Compression Methods, https://arxiv.org/abs/2407.09510v5, fetched 2026-10-26
[S3] EAGLES: Efficient Accelerated 3D Gaussians with Lightweight EncodingS, https://doi.org/10.1007/978-3-031-73036-8_4, fetched 2026-10-27
