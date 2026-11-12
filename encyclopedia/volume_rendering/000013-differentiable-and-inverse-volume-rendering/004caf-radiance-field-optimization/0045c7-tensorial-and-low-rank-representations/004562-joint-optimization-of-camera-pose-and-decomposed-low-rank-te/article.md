# Joint Optimization of Camera Pose and Decomposed Low-Rank Tensorial Radiance Fields

Also called: Joint TensoRF

## Overview
Joint TensoRF is a method for the joint refinement of camera poses and scene geometry represented by decomposed low-rank tensors, using only 2D image supervision [S1]. It solves the problem that naive joint optimization on voxel-based radiance fields leads to suboptimal solutions because voxel-based methods lack the spectral bias inherent in MLPs, causing them to overemphasize sharp edges and get trapped in local optima [S1]. Unlike [TensoRF](3026-tensorf.md), which assumes known camera poses, this method integrates efficient spectral filtering into the training loop to enable stable pose optimization [S1].

## How it works
**Inputs:** A set of 2D images with initial camera poses and intrinsics; a decomposed low-rank tensor representation of the radiance field (typically Vector-Matrix format as described in [Tensorial and Low-Rank Representations](11617-tensorial-and-low-rank-representations.md)).
**Outputs:** Refined camera poses and an optimized radiance field.

**Data Structures:**
The scene is represented by decomposed low-rank tensors. For density $\sigma$ and color features $\mathbf{c}$, the method uses mode vectors and matrices. The core innovation is the application of convolutional Gaussian filters on these fields.

**Algorithm Steps:**

1.  **Initialization:** Initialize the decomposed low-rank tensor factors and camera poses. The tensor factors are typically initialized at a base resolution $N_0^3$ [S1][3026-tensorf.md].

2.  **Spectral Filtering (Coarse-to-Fine):** To mitigate the lack of spectral bias in voxel-based representations, apply convolutional Gaussian filters to both the 2D supervision images and the 3D radiance field [S1].
    *   **Efficient 3D Convolution:** Instead of brute-force 3D convolution, leverage the decomposition property of the low-rank tensor. The convolution is applied to the individual mode vectors and matrices. This achieves the equivalent effect of a 3D convolution with minimal computational overhead [S1].
    *   **Filter Parameters:** Use Gaussian kernels with randomly scaled parameters during training to improve robustness [S1].

3.  **Rendering:** Render the filtered 2D images from the current camera poses using the differentiable volume rendering equation. The radiance field values are obtained by interpolating the tensor factors (trilinear interpolation for vectors/matrices as per [Tensorial and Low-Rank Representations](11617-tensorial-and-low-rank-representations.md)) [S1][3026-tensorf.md].

4.  **Loss Computation:** Compute the loss between the rendered images and the ground truth 2D images.
    *   **Smoothed 2D Supervision:** Apply smoothing to the supervision images to reduce high-frequency noise that could destabilize pose optimization [S1].
    *   **Edge-Guided Loss Mask:** Use an edge-guided loss mask to focus optimization on structural details, improving stability [S1].

5.  **Optimization:** Update both the camera poses and the tensor factors using gradient descent. The joint optimization is performed for 50,000 iterations [S1].

6.  **Upsampling:** If using a coarse-to-fine grid strategy, upsample the tensor factors at specific iteration steps (e.g., 2000, 3000, 4000) using linear interpolation for vectors and bilinear interpolation for matrices, as described in [TensoRF Reconstruction Optimization](3453-tensorf-reconstruction-optimization.md) [S1][3453-tensorf-reconstruction-optimization.md].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def joint_optimization(images, initial_poses, initial_tensor_factors, num_iterations=50000):
    """
    Jointly optimize camera poses and decomposed low-rank tensor radiance field.
    
    Args:
        images: List of ground truth 2D images.
        initial_poses: Initial camera poses (extrinsics).
        initial_tensor_factors: Initial decomposed low-rank tensor factors (vectors/matrices).
        num_iterations: Number of training iterations.
        
    Returns:
        optimized_poses: Refined camera poses.
        optimized_factors: Optimized tensor factors.
    """
    # Initialize state
    poses = initial_poses
    factors = initial_tensor_factors
    
    # Define loss functions and filters
    edge_mask_generator = EdgeGuidedLossMask()
    gaussian_filter = ConvolutionalGaussianFilter()
    
    for step in range(num_iterations):
        # 1. Apply spectral filtering to supervision images
        # Randomly scale kernel parameters for robustness [S1]
        filtered_images = []
        for img in images:
            kernel_scale = random_scale() # [G] Placeholder for random scaling logic
            filtered_img = gaussian_filter.apply(img, kernel_scale)
            filtered_images.append(filtered_img)
            
        # 2. Efficient 3D convolution on tensor factors
        # Leverage decomposition to avoid brute-force 3D conv [S1]
        # Apply Gaussian filter to each mode vector/matrix
        filtered_factors = []
        for factor in factors:
            # factor is either a vector or matrix
            filtered_factor = gaussian_filter.apply_on_tensor_factor(factor)
            filtered_factors.append(filtered_factor)
            
        # 3. Render images from current poses and filtered factors
        rendered_images = []
        for pose, filt_img in zip(poses, filtered_images):
            # Differentiable volume rendering using filtered factors
            # Interpolation is handled by the tensor factor structure [3026-tensorf.md]
            rendered_img = volume_render(filtered_factors, pose)
            rendered_images.append(rendered_img)
            
        # 4. Compute Loss
        # Smoothed 2D supervision and edge-guided mask [S1]
        total_loss = 0
        for rendered, ground_truth in zip(rendered_images, filtered_images):
            # Edge-guided mask
            mask = edge_mask_generator.compute(ground_truth)
            # L1 or L2 loss with mask
            loss = torch.mean(mask * torch.abs(rendered - ground_truth))
            total_loss += loss
            
        # 5. Backpropagation
        # Update poses and factors
        gradients_poses = torch.autograd.grad(total_loss, poses, retain_graph=True)[0]
        gradients_factors = torch.autograd.grad(total_loss, factors)
        
        poses -= learning_rate * gradients_poses
        factors = update_tensor_factors(factors, gradients_factors)
        
        # 6. Upsampling if necessary (coarse-to-fine)
        if step in [2000, 3000, 4000]: # Example steps [3453-tensorf-reconstruction-optimization.md]
            factors = upsample_tensor_factors(factors)
            
    return poses, factors
```

**Parameters:**
*   Training iterations: 50,000 [S1].
*   Previous methods typically require 200,000 iterations [S1].

**Edge Cases and Pitfalls:**
*   **Local Optima:** Naive joint optimization on voxel-based NeRFs easily leads to suboptimal solutions due to the lack of spectral bias, which causes the model to overemphasize sharp edges [S1]. The proposed Gaussian filtering and coarse-to-fine schedule are critical to avoid this [S1].
*   **Precision:** The efficiency relies on the decomposition property; if the rank is too high, the computational overhead of the "efficient" convolution may approach that of brute-force 3D convolution [S1].

## Operating characteristics
*   **Time Complexity:** The method reduces training time by 25% compared to previous MLP-based methods [S1]. This is attributed to the efficient spectral filtering algorithm which requires only a single reusable voxel grid [S1].
*   **Memory Complexity:** Inherits the low memory usage of decomposed low-rank tensors (TensoRF) [S1].
*   **Performance:** Achieves superior performance in novel view synthesis and rapid convergence [S1].
*   **Strengths:** Robust joint pose refinement; efficient computation via separable component-wise convolution; reduced training iterations (50k vs 200k) [S1].
*   **Limitations:** Requires careful tuning of filter parameters and edge-guided masks for stability [S1].

## Background
Joint TensoRF was proposed by Bo-Yu Cheng, Wei-Chen Chiu, and Yu-Lun Liu from National Yang Ming Chiao Tung University [S1]. It addresses the limitations of voxel-based NeRFs in joint pose optimization, which lack the spectral bias of MLPs [S1]. The method builds on the TensoRF framework [3026-tensorf.md] and incorporates techniques from [TensoRF Reconstruction Optimization](3453-tensorf-reconstruction-optimization.md) for upsampling. The source code is available at https://github.com/Nemo1999/Joint-TensoRF [S1].

## Sources
[S1] Improving Robustness for Joint Optimization of Camera Pose and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.1609/aaai.v38i2.27859, fetched 2026-10-26
