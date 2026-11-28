# Laplacian Frequency Hierarchies

Also called: Laplacian Frequency Hierarchies for 3D Gaussian Splatting, Laplacian framework, Laplacian-GS, coarse-to-fine, frequency-staged training, frequency-factorized 3DGS

## Overview
Laplacian Frequency Hierarchies is a training optimization method for 3D Gaussian Splatting that decomposes the image supervision signal into low-frequency and high-frequency components [S1]. The method trains separate Gaussian fields for each component in a coarse-to-fine sequence [S1]. Unlike prior frequency-based methods that use frequency cues only for scheduling or resolution adjustment, this approach incorporates frequency at the representation level by factorizing the scene into frequency-specific fields [S1]. Completed lower-frequency fields are archived to reduce the active Gaussian count during subsequent high-frequency optimization [S1]. The framework serves as a plug-and-play module that reduces training time and peak memory usage while maintaining competitive reconstruction quality, particularly at high resolutions [S1].

## How it works
**Inputs and Outputs**
The method takes a set of calibrated input images with corresponding camera poses, a target resolution (e.g., 1K, 2K, 4K), a total iteration budget, and a backbone 3DGS implementation as inputs [S1]. It outputs a composed novel-view synthesis image and a set of Gaussian primitives distributed across $L$ frequency levels [S1].

**Data Structures**
The method maintains $L$ distinct Gaussian fields $\mathcal{G}_1, \dots, \mathcal{G}_L$, where each field contains primitives parameterized by a mean position $\mu_i \in \mathbb{R}^3$, an opacity $\sigma_i \in (0, 1)$, a covariance matrix $\Sigma_i \in \mathbb{R}^{3\times3}$, and view-dependent appearance parameters [S1]. The covariance matrix is parameterized as $\Sigma_i = R_i S_i S_i^\top R_i^\top$ with rotation $R_i$ and diagonal scale $S_i = \text{diag}(s_i)$ [S1]. The Gaussian density function is defined as $G_i(x) = \exp\left(-\frac{1}{2}(x - \mu_i)^\top \Sigma_i^{-1}(x - \mu_i)\right)$ where $x$ is a 3D point [S1]. Each field is rendered independently to produce a frequency-band image $R_{v}^\ell$ [S1]. The final image is the sum of these components: $\hat{I}_v = \sum_{\ell=1}^{L} R_{v}^\ell$ [S1].

**Number of Levels and Iteration Split**
The hierarchy depth $L$ depends on the target resolution [S1]. For 1K resolution, the default is $L=2$ with an iteration split of $(10,000, 20,000)$ [S1]. For 4K resolution, the default is $L=4$ with an iteration split of $(15,000, 7,500, 7,500, 15,000)$ [S1]. Increasing $L$ beyond these defaults consistently degrades reconstruction quality due to accumulated structural misalignment errors across frequency bands [S1].

**Algorithm Steps**
1. **Initialize:** Create $L$ empty Gaussian fields $\mathcal{G}_1, \dots, \mathcal{G}_L$ [S1]. Initialize $\mathcal{G}_1$ with standard 3DGS initialization from a structure-from-motion point cloud [G].
2. **Iterate $\ell$ from 1 to $L$:**
   a. **Active Set:** The active Gaussian set for optimization is $\mathcal{G}_\ell$ [S1]. All previous fields $\mathcal{G}_1, \dots, \mathcal{G}_{\ell-1}$ are archived, meaning they are frozen, gradients are not computed, and density control operations are disabled [S1].
   b. **Render:** For each training view $v$, render the current field $\mathcal{G}_\ell$ to obtain the band-specific image $R_{v}^\ell$ using standard 3DGS rasterization [S1].
   c. **Loss:** Compute the photometric loss between the rendered image $R_{v}^\ell$ and the ground truth image $I_v$ [S1]. The method uses reconstructed-image supervision, meaning the loss is computed against the full ground truth image rather than the residual of the ground truth [S1]. The loss is $\mathcal{L}_\ell = \mathcal{L}_{photo}(R_{v}^\ell, I_v)$ [S1]. Directly fitting the residual underperforms this default design [S1].
   d. **Optimize:** Perform gradient descent on the parameters of $\mathcal{G}_\ell$ using the loss $\mathcal{L}_\ell$ [S1]. Apply standard 3DGS density control (clone, split, prune) only to $\mathcal{G}_\ell$ [S1].
   e. **Archive:** After the allocated iterations for level $\ell$ are complete, freeze $\mathcal{G}_\ell$ [S1].
3. **Inference:** To render a novel view, sequentially render each archived field $\mathcal{G}_1, \dots, \mathcal{G}_L$ and sum the results pixel-wise [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import random

def train_laplacian_gs(input_images, camera_poses, target_resolution, total_iterations, num_levels):
    """
    Train 3D Gaussian Splatting using Laplacian Frequency Hierarchies.
    """
    # Determine iteration split based on resolution and levels [S1]
    if target_resolution == '1K' and num_levels == 2:
        iterations_per_level = [10000, 20000]
    elif target_resolution == '4K' and num_levels == 4:
        iterations_per_level = [15000, 7500, 7500, 15000]
    else:
        iterations_per_level = [total_iterations // num_levels] * num_levels

    fields = []
    # Initialize first field with standard 3DGS initialization from SfM point cloud [G]
    current_field = [{'mu': p, 'sigma': 0.5, 'Sigma': np.eye(3), 'appearance': np.zeros(32)} for p in input_images[0].points]
    fields.append(current_field)

    for level in range(num_levels):
        field = fields[-1]
        n_iters = iterations_per_level[level]

        for step in range(n_iters):
            view_idx = random.randint(0, len(input_images) - 1)
            gt_image = input_images[view_idx].data
            pose = camera_poses[view_idx]

            # Render current field using standard 3DGS affine projection and alpha blending [G]
            rendered_band = render_gaussian_field(field, pose)

            # Compute photometric loss against full ground truth image [S1]
            loss = np.mean(np.abs(rendered_band - gt_image))

            # Update field parameters using gradient descent [G]
            for prim in field:
                prim.mu -= 0.01 * np.gradient(loss, prim.mu)
                prim.Sigma -= 0.01 * np.gradient(loss, prim.Sigma)

        # Archive field: freeze parameters and stop density control [S1]
        field.frozen = True

    return fields

def render_gaussian_field(field, pose):
    """
    Standard 3DGS rasterization: project 3D Gaussians to 2D ellipses,
    sort by depth, and alpha-blend across tiles [G].
    """
    projected_gaussians = []
    for prim in field:
        center_2d = pose.projection(prim.mu)
        cov_2d = pose.project_covariance(prim.Sigma)
        projected_gaussians.append({'center': center_2d, 'cov': cov_2d, 'sigma': prim.sigma, 'appearance': prim.appearance})
    image = np.zeros((pose.height, pose.width, 3))
    for g in sorted(projected_gaussians, key=lambda x: x['depth']):
        image = alpha_blend(image, g)
    return image

def alpha_blend(image, gaussian):
    """
    Rasterize a single 2D Gaussian ellipse and blend it into the image buffer.
    """
    # Compute ellipse bounds and sample pixels
    for y, x in ellipse_pixels(gaussian['center'], gaussian['cov']):
        if 0 <= y < image.shape[0] and 0 <= x < image.shape[1]:
            weight = np.exp(-0.5 * (gaussian['cov'].inv @ np.array([x, y]).T).sum())
            alpha = gaussian['sigma'] * weight
            image[y, x] = image[y, x] * (1 - alpha) + gaussian['appearance'] * alpha
    return image

def ellipse_pixels(center, cov):
    """
    Generate pixel coordinates within the bounding box of a 2D Gaussian covariance matrix.
    """
    # Standard bounding box calculation from covariance eigenvalues
    return [(y, x) for y in range(int(center[1]-2), int(center[1]+2)) for x in range(int(center[0]-2), int(center[0]+2))]
```

**Edge Cases and Pitfalls**
Increasing the number of Laplacian levels $L$ consistently degrades reconstruction quality due to the accumulation of structural misalignment errors across multiple frequency bands [S1]. Directly fitting the residual by using the residual of the ground truth as the target underperforms the default reconstructed-image supervision [S1]. The method must supervise the final reconstructed image against the ground truth at each level [S1]. Inference FPS may decrease compared to single-field baselines due to sequential rendering of multiple frequency fields, which should be considered when latency is critical [S1].

## Operating characteristics
**Time and Memory Complexity**
The method significantly reduces training time by splitting the workload into smaller Gaussian fields [S1]. Average speedups of $1.73\times$ and $1.21\times$ at 1K resolution, and $1.74\times$ and $1.33\times$ at 4K resolution are reported when combined with Taming-3DGS and FastGS backbones, respectively [S1]. By archiving lower-frequency fields and training higher-frequency fields with fewer active primitives, the peak GPU memory usage and peak Gaussian count are reduced [S1]. For example, in a 4K setting, the peak Gaussian count is reduced from 1.74M to 0.49M [S1].

**Measured Performance**
On the MipNeRF 360 1K setting with a Taming-3DGS backbone, training time is reduced from 8.88 to 5.13 minutes ($1.73\times$ speedup) with a 0.34 dB PSNR drop [S1]. On the same setting with a FastGS backbone, training time is reduced from 4.82 to 3.96 minutes with a 0.29 dB PSNR drop [S1]. Efficiency gains become more pronounced at higher resolutions, with larger speedups while maintaining comparable reconstruction quality [S1].

**Strengths and Limitations**
The method offers plug-and-play compatibility with existing 3DGS accelerations like Taming-3DGS and FastGS [S1]. It provides a significant reduction in training time and peak memory usage, and scales well to high resolutions such as 2K and 4K [S1]. Limitations include a potential decrease in inference FPS due to sequential rendering of multiple fields, slight metric degradation from approximation errors in hierarchical composition, and quality degradation when using deeper hierarchies than recommended [S1].

**When to Use**
The method should be used when training time and GPU memory are constrained, especially at high resolutions [S1]. It is also suitable when integrating with accelerated backbones like Taming-3DGS or FastGS [S1]. Inference FPS may decrease compared to single-field baselines due to sequential rendering of multiple frequency fields, which should be considered when latency is critical [S1].

## Background
Laplacian Frequency Hierarchies was proposed by Yixiong Yang, Sisheng Zhang, Qingsong Yan, Shaohuai Shi, and Qiang Wang from Harbin Institute of Technology, Shenzhen, and XGRIDS [S1]. The paper, titled "Laplacian Frequency Hierarchies for Efficient 3D Gaussian Splatting Training," was published on arXiv in September 2026 [S1]. The work addresses the bottleneck of continual Gaussian growth and heavy optimization costs in 3DGS training, particularly at high resolutions [S1]. It builds on the observation that image-plane signals have naturally structured frequency components, leveraging Laplacian decomposition to factorize the representation [S1].

## Sources
[S1] Laplacian Frequency Hierarchies for Efficient 3D Gaussian Splatting Training, https://doi.org/10.48550/arxiv.2609.03334, fetched 2026-10-26
