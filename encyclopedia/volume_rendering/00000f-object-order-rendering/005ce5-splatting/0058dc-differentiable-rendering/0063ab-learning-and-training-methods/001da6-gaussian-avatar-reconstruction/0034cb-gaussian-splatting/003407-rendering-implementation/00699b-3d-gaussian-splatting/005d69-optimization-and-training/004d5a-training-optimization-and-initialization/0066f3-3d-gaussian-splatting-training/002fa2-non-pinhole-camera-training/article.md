# Non-Pinhole Camera Training

Also called: Non-Pinhole Camera Adaptation

## Overview
Non-Pinhole Camera Training extends 3D Gaussian Splatting to handle camera models that deviate from the standard pinhole assumption, such as fisheye lenses or uncentered projections. It solves the problem of geometric distortion and coordinate misalignment in non-standard imaging systems by integrating differentiable image-space warping and masking into the training loop. It sets itself apart from standard 3DGS by explicitly supporting uncentered-distorted and uncentered-undistorted camera models, whereas standard 3DGS assumes a centered pinhole model [S3].

## How it works
The method adapts the differentiable rasterizer to accommodate various non-pinhole camera geometries by modifying the projection pipeline. The core requirement is that all coordinate transformations and pixel masking operations must remain differentiable to enable end-to-end gradient backpropagation [S3].

### Inputs and Outputs
- **Inputs**: Training images from non-pinhole cameras (e.g., fisheye, wide-angle), corresponding camera parameters (intrinsics, distortion coefficients, center offsets), and initial 3D Gaussian attributes (position, covariance, opacity, spherical harmonics).
- **Outputs**: Optimized 3D Gaussian representation and camera parameters that reconstruct the scene under the specific non-pinhole projection.

### Data Structures
The Gaussian attributes remain standard (position $\mathbf{p}$, rotation $\mathbf{q}$, scale $\mathbf{s}$, opacity $\alpha$, SH coefficients). The camera model structure is extended to include:
- **Center Offset**: $(c_x, c_y)$ for uncentered models.
- **Distortion Coefficients**: $k_1, k_2, \dots$ for radial/tangential distortion.
- **Warping Function**: A differentiable mapping $W: \mathbb{R}^2 \to \mathbb{R}^2$ that maps ideal perspective coordinates to distorted image coordinates.

### Steps
1. **Camera Model Selection**: Choose the appropriate camera model based on the dataset:
   - **Centered-Undistorted**: Standard pinhole model (used for Neural 3D Video Dataset) [S3].
   - **Uncentered-Undistorted**: Pinhole model with principal point offset (used for Technicolor Dataset) [S3].
   - **Uncentered-Distorted**: Fisheye or wide-angle model with distortion and offset (used for Google Immersive Dataset) [S3].
2. **Differentiable Warping**: For distorted models, apply a differentiable image-space warping function $W$ to map perspective view coordinates to the distorted image plane. This function must support gradient computation [S3].
3. **Masking**: Generate a binary mask $M$ to exclude invalid pixels (e.g., black borders resulting from undistortion or out-of-bounds warping). The mask must be differentiable or applied in a way that allows gradients to flow through valid pixels [S3].
4. **Rasterization**: Render the Gaussians using the adapted projection. The rasterizer computes the contribution of each Gaussian to the image plane, accounting for the warped coordinates and masked regions.
5. **Loss Computation**: Compute the rendering loss (e.g., L1, SSIM) only over valid pixels where $M=1$.
6. **Optimization**: Update Gaussian attributes and camera parameters using gradient descent. The differentiability of $W$ and $M$ ensures gradients propagate correctly from the loss to the Gaussians [S3].

### Formulas
- **Warping**: $\mathbf{u}_{distorted} = W(\mathbf{u}_{perspective})$, where $\mathbf{u}$ are 2D image coordinates.
- **Masking**: $\mathbf{I}_{rendered} = \mathbf{I}_{rendered} \odot M$, where $\odot$ is element-wise multiplication.
- **Loss**: $\mathcal{L} = \sum_{i,j} M_{i,j} \cdot \|\mathbf{I}_{rendered}(i,j) - \mathbf{I}_{target}(i,j)\|$.

### Parameters
- **Distortion Coefficients**: Typically $k_1, k_2$ for radial distortion. Values depend on the lens.
- **Center Offset**: $(c_x, c_y)$ determined by camera calibration.
- **Mask Threshold**: Pixels with intensity below a threshold (e.g., 0) are masked out [S3].

### Edge Cases and Pitfalls
- **Non-Differentiable Masking**: If the mask is not differentiable, gradients may be blocked. Use soft masks or ensure the warping function handles boundaries smoothly [S3].
- **Black Pixels**: Undistorted images often have black borders. These must be masked out to prevent loss computation on invalid data [S3].
- **Coordinate System Mismatch**: Ensure the warping function aligns with the rasterizer's coordinate system (e.g., origin at center vs. corner) [S3].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_non_pinhole(gaussians, camera, image_size):
    # 1. Project Gaussians to perspective plane
    perspective_coords = project_to_perspective(gaussians, camera)
    
    # 2. Apply differentiable warping for distorted models
    if camera.is_distorted:
        distorted_coords = differentiable_warp(perspective_coords, camera.distortion_coeffs)
    else:
        distorted_coords = perspective_coords
        
    # 3. Rasterize using distorted coordinates
    rendered_image = rasterize(gaussians, distorted_coords, image_size)
    
    # 4. Apply differentiable mask for invalid pixels
    mask = generate_differentiable_mask(distorted_coords, image_size)
    masked_image = rendered_image * mask
    
    # 5. Compute loss against target
    loss = compute_loss(masked_image, target_image, mask)
    
    return loss
```

## Subtopics

### Aperture Randomized Training
A training technique introduced to disentangle defocus-aware and defocus-independent representations in an unsupervised manner by learning to generate images while randomizing the aperture size and latent codes independently [S2]. During training, the aperture size is changed randomly, and the latent codes are changed randomly [S2]. This technique is similar to techniques commonly used in viewpoint-aware representation learning, which are useful for disentangling the effect of the corresponding factor from latent codes [S2].

### Non-Pinhole Camera Adaptation
Adaptation of the 3DGS pipeline to specific non-pinhole datasets. For the Neural 3D Video Dataset, use the original centered-undistorted camera model [S3]. For the Technicolor Dataset, implement and use an uncentered-undistorted camera model [S3]. For the Google Immersive Dataset, adapt the method to fit an uncentered-distorted camera model using a differentiable image space warping function that maps perspective views to fish-eye distorted views [S3]. For real-time demos, retrain models on undistorted videos and apply a differentiable mask to exclude black pixels resulting from undistortion [S3]. Image warping and masking must remain differentiable to allow end-to-end training despite the coordinate transformations and pixel masking [S3].

## Operating characteristics
- **Time Complexity**: Similar to standard 3DGS, but with additional overhead for warping and masking operations. The differentiable warping adds computational cost per pixel [S3].
- **Memory Complexity**: No significant increase in memory usage compared to standard 3DGS, as the Gaussian attributes remain the same [S3].
- **Performance**: Successfully reconstructs scenes on non-pinhole datasets where standard 3DGS fails entirely, such as the Matterport dataset [S1].
- **Strengths**: Handles a wide variety of camera models, including fisheye and uncentered projections. Enables training on real-world data with non-standard lenses [S3].
- **Limitations**: Requires careful implementation of differentiable warping and masking. Performance may degrade if the warping function is not smooth or if the mask is not properly handled [S3].
- **When to Use**: When training on datasets with non-pinhole cameras (e.g., fisheye, wide-angle, uncentered) [S3].
- **When Not to Use**: For standard pinhole camera datasets, where standard 3DGS is sufficient and simpler [S3].

## Background
Non-Pinhole Camera Training was developed to address the limitations of standard 3DGS in handling non-standard camera models. It builds on the differentiable rasterizer of 3DGS, extending it to support various camera geometries [S3]. The method has been applied to datasets such as the Neural 3D Video Dataset, Technicolor Dataset, and Google Immersive Dataset [S3]. It has also been shown to succeed on the Matterport dataset, where standard 3DGS fails [S1]. The technique of Aperture Randomized Training, while related to defocus disentanglement, shares the principle of randomizing camera parameters during training to improve representation learning [S2].

## Sources
[S1] On the Error Analysis of 3D Gaussian Splatting and an Optimal Projection Strategy, https://doi.org/10.1007/978-3-031-72643-9_15, fetched 2026-10-29
[S2] AR-NeRF: Unsupervised Learning of Depth and Defocus Effects from Natural Images with Aperture Rendering Neural Radiance Fields, https://arxiv.org/abs/2206.06100v1, fetched 2026-10-26
[S3] Spacetime Gaussian Feature Splatting for Real-Time Dynamic View Synthesis, https://huggingface.co/papers/2312.16812, fetched 2026-10-30
