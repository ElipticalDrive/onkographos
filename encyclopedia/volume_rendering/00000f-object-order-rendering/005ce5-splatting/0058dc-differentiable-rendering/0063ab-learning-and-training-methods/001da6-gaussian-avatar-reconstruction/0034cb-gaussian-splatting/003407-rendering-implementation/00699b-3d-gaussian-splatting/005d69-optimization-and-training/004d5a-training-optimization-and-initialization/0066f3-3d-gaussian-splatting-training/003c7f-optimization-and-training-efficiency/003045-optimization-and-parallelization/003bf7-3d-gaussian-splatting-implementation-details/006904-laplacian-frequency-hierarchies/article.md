# Laplacian Frequency Hierarchies

Also called: Laplacian Frequency Hierarchies for 3D Gaussian Splatting, Laplacian framework, Laplacian-GS, coarse-to-fine, frequency-staged training, frequency-factorized 3DGS

## Overview
Laplacian Frequency Hierarchies is a training optimization method for 3D Gaussian Splatting (3DGS) that decomposes the image supervision signal into low-frequency and high-frequency components, training separate Gaussian fields for each component in a coarse-to-fine sequence [S1]. Unlike prior frequency-based methods that use frequency cues only for scheduling or resolution adjustment, this method incorporates frequency at the representation level by factorizing the scene into frequency-specific fields and archiving completed lower-frequency fields to reduce the active Gaussian count during subsequent high-frequency optimization [S1]. It serves as a plug-and-play module that reduces training time and peak memory usage while maintaining competitive reconstruction quality, particularly at high resolutions [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** A set of calibrated input images $\{I_v\}$ with corresponding camera poses; a target resolution (e.g., 1K, 2K, 4K); a total iteration budget $T_{total}$; and a backbone 3DGS implementation (e.g., Taming-3DGS or FastGS) [S1].
*   **Outputs:** A composed novel-view synthesis image $\hat{I}_v$ and a set of Gaussian primitives $\mathcal{G}$ distributed across $L$ frequency levels [S1].

**Data Structures**
The method maintains $L$ distinct Gaussian fields $\mathcal{G}_1, \dots, \mathcal{G}_L$, where each field $\mathcal{G}_\ell$ contains primitives parameterized by:
*   Mean position $\mu_i \in \mathbb{R}^3$
*   Opacity $\sigma_i \in (0, 1)$
*   Covariance matrix $\Sigma_i \in \mathbb{R}^{3\times3}$, parameterized as $\Sigma_i = R_i S_i S_i^\top R_i^\top$ with rotation $R_i$ and diagonal scale $S_i = \text{diag}(s_i)$ [S1]
*   View-dependent appearance parameters (typically spherical harmonics coefficients) [S1]

Each field is rendered independently to produce a frequency-band image $R_{v}^\ell$. The final image is the sum of these components:
$$ \hat{I}_v = \sum_{\ell=1}^{L} R_{v}^\ell $$
[S1]

**Number of Levels and Iteration Split**
The hierarchy depth $L$ depends on the target resolution:
*   **1K Resolution:** Default $L=2$. Iteration split is $(10,000, 20,000)$ [S1].
*   **4K Resolution:** Default $L=4$. Iteration split is $(15,000, 7,500, 7,500, 15,000)$ [S1].
Increasing $L$ beyond these defaults consistently degrades reconstruction quality due to accumulated structural misalignment errors across frequency bands [S1].

**Algorithm Steps**
1.  **Initialize:** Create $L$ empty Gaussian fields $\mathcal{G}_1, \dots, \mathcal{G}_L$. Initialize $\mathcal{G}_1$ with standard 3DGS initialization (e.g., point cloud from SfM) [S1].
2.  **Iterate $\ell$ from 1 to $L$:**
    a.  **Active Set:** The active Gaussian set for optimization is $\mathcal{G}_\ell$. All previous fields $\mathcal{G}_1, \dots, \mathcal{G}_{\ell-1}$ are "archived" (frozen, no gradients computed, no densification/culling) [S1].
    b.  **Render:** For each training view $v$, render the current field $\mathcal{G}_\ell$ to obtain the residual image $R_{v}^\ell$ using standard 3DGS rasterization [S1].
    c.  **Loss:** Compute the photometric loss between the rendered image $R_{v}^\ell$ and the ground truth image $I_v$. Note: The method uses **reconstructed-image supervision**, meaning the loss is computed against the full ground truth image, not the residual of the ground truth [S1].
        $$ \mathcal{L}_\ell = \mathcal{L}_{photo}(R_{v}^\ell, I_v) $$
        [S1]
        *Pitfall:* Supervising the residual directly (i.e., $\mathcal{L}_{photo}(R_{v}^\ell, I_v - \sum_{k<\ell} R_{v}^k)$) underperforms the default design [S1].
    d.  **Optimize:** Perform gradient descent on the parameters of $\mathcal{G}_\ell$ using the loss $\mathcal{L}_\ell$. Apply standard 3DGS density control (clone, split, prune) only to $\mathcal{G}_\ell$ [S1].
    e.  **Archive:** After the allocated iterations for level $\ell$ are complete, freeze $\mathcal{G}_\ell$.
3.  **Inference:** To render a novel view, sequentially render each archived field $\mathcal{G}_1, \dots, \mathcal{G}_L$ and sum the results pixel-wise [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_laplacian_gs(input_images, camera_poses, target_resolution, total_iterations, num_levels):
    """
    Train 3D Gaussian Splatting using Laplacian Frequency Hierarchies.
    
    Args:
        input_images: List of ground truth images.
        camera_poses: List of camera poses corresponding to images.
        target_resolution: Resolution string (e.g., '1K', '4K').
        total_iterations: Total number of training iterations.
        num_levels: Number of Laplacian levels L.
        
    Returns:
        archived_fields: List of frozen Gaussian fields [G_1, ..., G_L].
    """
    
    # Determine iteration split based on resolution and levels
    # Default splits are hardcoded based on source evidence [S1]
    if target_resolution == '1K' and num_levels == 2:
        iterations_per_level = [10000, 20000]
    elif target_resolution == '4K' and num_levels == 4:
        iterations_per_level = [15000, 7500, 7500, 15000]
    else:
        # Fallback: equal split if not specified, though source notes 
        # that performance is insensitive to split [S1]
        iterations_per_level = [total_iterations // num_levels] * num_levels

    # Initialize empty fields
    fields = []
    
    # Initialize first field with standard 3DGS initialization
    # [G] Standard 3DGS initialization from SfM point cloud is assumed here
    current_field = initialize_3dgs_fields(input_images, camera_poses)
    fields.append(current_field)
    
    current_iter = 0
    
    for level in range(num_levels):
        field = fields[-1]
        n_iters = iterations_per_level[level]
        
        # Optimization loop for the current frequency level
        for step in range(n_iters):
            # Sample a random training view
            view_idx = random_sample(input_images)
            gt_image = input_images[view_idx]
            pose = camera_poses[view_idx]
            
            # Render the current field to get the band-specific image
            # This renders ONLY the Gaussians in the current field
            rendered_band = render_gaussian_field(field, pose)
            
            # Compute loss against the FULL ground truth image
            # Source [S1] specifies reconstructed-image supervision
            loss = compute_photometric_loss(rendered_band, gt_image)
            
            # Backpropagate and update ONLY the current field's parameters
            # Standard 3DGS density control (clone/split/prune) applies here
            optimize_field(field, loss)
            
            current_iter += 1
            
        # Archive the field: freeze parameters, stop tracking gradients
        field.freeze()
        
        # If not the last level, initialize the next field
        if level < num_levels - 1:
            # [G] The next field is initialized empty or with sparse points.
            # Source [S1] implies subsequent fields optimize residuals without 
            # carrying the full primitive burden, suggesting they start small.
            next_field = initialize_empty_field()
            fields.append(next_field)
            
    return fields

def render_novel_view(archived_fields, camera_pose):
    """
    Compose the final image by summing rendered frequency bands.
    """
    final_image = zeros_like(camera_pose.resolution)
    
    for field in archived_fields:
        band_image = render_gaussian_field(field, camera_pose)
        final_image += band_image
        
    return final_image
```

**Edge Cases and Pitfalls**
*   **Depth of Hierarchy:** Increasing the number of Laplacian levels $L$ consistently degrades reconstruction quality [S1]. This is attributed to the accumulation of structural misalignment errors across multiple frequency bands [S1].
*   **Supervision Signal:** Directly fitting the residual (using the residual of the ground truth as the target) underperforms the default reconstructed-image supervision [S1]. The method must supervise the final reconstructed image against the ground truth at each level [S1].
*   **Inference Speed:** While training is accelerated, inference FPS may decrease compared to a single-field baseline because the final image requires sequential rendering of multiple frequency fields [S1].

## Operating characteristics
**Time and Memory Complexity**
*   **Training Time:** The method significantly reduces training time by splitting the workload into smaller Gaussian fields. Average speedups of $1.73\times$ and $1.21\times$ at 1K resolution, and $1.74\times$ and $1.33\times$ at 4K resolution are reported when combined with Taming-3DGS and FastGS backbones, respectively [S1].
*   **Memory Usage:** By archiving lower-frequency fields and training higher-frequency fields with fewer active primitives, the peak GPU memory usage and peak Gaussian count are reduced. For example, in a 4K setting, the peak Gaussian count is reduced from 1.74M to 0.49M [S1].

**Measured Performance**
*   **1K Resolution (MipNeRF 360):**
    *   With Taming-3DGS backbone: Training time reduced from 8.88 to 5.13 minutes ($1.73\times$ speedup) with a 0.34 dB PSNR drop [S1].
    *   With FastGS backbone: Training time reduced from 4.82 to 3.96 minutes with a 0.29 dB PSNR drop [S1].
*   **4K Resolution:** Efficiency gains become more pronounced at higher resolutions, with larger speedups while maintaining comparable reconstruction quality [S1].

**Strengths and Limitations**
*   **Strengths:** Plug-and-play compatibility with existing 3DGS accelerations (Taming-3DGS, FastGS); significant reduction in training time and peak memory; scalable to high resolutions (2K, 4K) [S1].
*   **Limitations:** Potential decrease in inference FPS due to sequential rendering of multiple fields; slight metric degradation (PSNR/SSIM) due to approximation errors from hierarchical composition; deeper hierarchies ($L>2$ for 1K) degrade quality [S1].

**When to Use**
*   Use when training time and GPU memory are constrained, especially at high resolutions (2K, 4K) [S1].
*   Use when integrating with accelerated backbones like Taming-3DGS or FastGS [S1].
*   Do not use if inference latency is the primary constraint and the scene is low-resolution, as the sequential rendering overhead may negate benefits [S1].

## Background
Laplacian Frequency Hierarchies was proposed by Yixiong Yang, Sisheng Zhang, Qingsong Yan, Shaohuai Shi, and Qiang Wang from Harbin Institute of Technology, Shenzhen, and XGRIDS [S1]. The paper, titled "Laplacian Frequency Hierarchies for Efficient 3D Gaussian Splatting Training," was published on arXiv in September 2026 [S1]. The work addresses the bottleneck of continual Gaussian growth and heavy optimization costs in 3DGS training, particularly at high resolutions [S1]. It builds on the observation that image-plane signals have naturally structured frequency components, leveraging Laplacian decomposition to factorize the representation [S1].

## Sources
[S1] Laplacian Frequency Hierarchies for Efficient 3D Gaussian Splatting Training, https://doi.org/10.48550/arxiv.2609.03334, fetched 2026-10-26
