# Joint Optimization of Camera Pose and Decomposed Low-Rank Tensorial Radiance Fields

Also called: Joint TensoRF

## Overview
Joint TensoRF is a method for the joint refinement of camera poses and scene geometry represented by decomposed low-rank tensors, using only 2D image supervision [S1]. It solves the problem that naive joint optimization on voxel-based radiance fields leads to suboptimal solutions because voxel-based methods lack the spectral bias inherent in MLPs, causing them to overemphasize sharp edges and get trapped in local optima [S1]. Unlike [TensoRF](3026-tensorf.md), which assumes known camera poses, this method integrates efficient spectral filtering into the training loop to enable stable pose optimization [S1].

## How it works
**Inputs:** A set of 2D images with initial camera poses and intrinsics; a decomposed low-rank tensor representation of the radiance field [S1].
**Outputs:** Refined camera poses and an optimized radiance field [S1].

**Data Structures:**
The scene is represented by decomposed low-rank tensors [S1]. For density $\sigma$ and color features $\mathbf{c}$, the method uses mode vectors and matrices [G]. The core mechanism applies convolutional Gaussian filters on these fields to control the frequency spectrum during training [S1].

**Algorithm Steps:**

1.  **Initialization:** Initialize the decomposed low-rank tensor factors and camera poses [G].
2.  **Spectral Filtering (Coarse-to-Fine):** Apply convolutional Gaussian filters to both the 2D supervision images and the 3D radiance fields to enable a coarse-to-fine training schedule [S1].
    *   **Efficient 3D Convolution:** Leverage the decomposition property of the low-rank tensor to achieve an equivalent effect to brute-force 3D convolution with minimal computational overhead [S1]. The convolution is applied to the individual mode vectors and matrices [S1].
    *   **Filter Parameters:** Use Gaussian kernels with randomly scaled parameters during training to improve robustness [S1].
3.  **Rendering:** Render the filtered 2D images from the current camera poses using differentiable volume rendering [G].
4.  **Loss Computation:** Compute the loss between the rendered images and the ground truth 2D images [G].
    *   **Smoothed 2D Supervision:** Apply smoothing to the supervision images to reduce high-frequency noise that could destabilize pose optimization [S1].
    *   **Edge-Guided Loss Mask:** Use an edge-guided loss mask to focus optimization on structural details, improving stability [S1].
5.  **Optimization:** Update both the camera poses and the tensor factors using gradient descent [G]. The joint optimization is performed for 50,000 iterations [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def apply_gaussian_2d(image, kernel_size, sigma):
    """Apply 2D Gaussian filter for smoothed supervision [S1]."""
    kernel = [math.exp(-0.5 * ((i - kernel_size//2)**2) / sigma**2) for i in range(kernel_size)]
    blurred = []
    for row in image:
        blurred_row = []
        for j in range(len(row)):
            acc = sum(row[max(0, j-kernel_size//2):min(len(row), j+kernel_size//2+1)] * 
                      kernel[max(0, kernel_size//2-j):min(len(kernel), kernel_size//2+1-j)])
            blurred_row.append(acc)
        blurred.append(blurred_row)
    return blurred

def apply_gaussian_3d_separable(factors, kernel_size, sigma):
    """Leverage decomposition to apply 3D Gaussian filter efficiently [S1]."""
    kernel = [math.exp(-0.5 * ((i - kernel_size//2)**2) / sigma**2) for i in range(kernel_size)]
    filtered_factors = []
    for factor in factors:
        if len(factor.shape) == 1:
            filtered_factors.append([sum(factor[max(0, i-kernel_size//2):min(len(factor), i+kernel_size//2+1)] * 
                                         kernel[max(0, kernel_size//2-i):min(len(kernel), kernel_size//2+1-i)] for i in range(len(factor)))])
        else:
            filtered_factors.append([[sum(row[max(0, j-kernel_size//2):min(len(row), j+kernel_size//2+1)] * 
                                         kernel[max(0, kernel_size//2-j):min(len(kernel), kernel_size//2+1-j)] for j in range(len(row))) for row in factor]])
    return filtered_factors

def compute_edge_mask(image):
    """Generate edge-guided loss mask [S1]."""
    return [[1.0 if abs(image[i][j] - image[i][j+1]) > 0.1 else 0.0 for j in range(len(image[0])-1)] for i in range(len(image))]

def volume_render(factors, pose):
    """Differentiable volume rendering using interpolated tensor factors [G]."""
    return [sum(f * p for f, p in zip(factors, pose)) / len(factors)]

def joint_optimization(images, initial_poses, initial_factors, num_iterations=50000):
    poses = initial_poses
    factors = initial_factors
    lr = 0.01  # [G] Standard learning rate
    kernel_size = 3
    base_sigma = 1.0

    for step in range(num_iterations):
        # 1. Spectral Filtering & Smoothed Supervision
        current_sigma = base_sigma * (0.5 + 0.5 * random.random())
        filtered_images = [apply_gaussian_2d(img, kernel_size, current_sigma) for img in images]

        # 2. Efficient 3D Convolution on Tensor Factors
        filtered_factors = apply_gaussian_3d_separable(factors, kernel_size, current_sigma)

        # 3. Rendering
        rendered_images = [volume_render(filtered_factors, pose) for pose in poses]

        # 4. Loss Computation with Edge-Guided Mask
        total_loss = 0.0
        for rendered, gt in zip(rendered_images, filtered_images):
            mask = compute_edge_mask(gt)
            total_loss += sum((r - g)**2 for r, g in zip(rendered, gt)) / len(rendered)

        # 5. Optimization
        grads_poses = [lr * random.random() for _ in poses]  # [G] Gradient computation placeholder
        grads_factors = [lr * random.random() for _ in factors]
        poses = [p - g for p, g in zip(poses, grads_poses)]
        factors = [f - g for f, g in zip(factors, grads_factors)]

    return poses, factors
```

**Parameters:**
*   Training iterations: 50,000 [S1].
*   Previous methods typically require 200,000 iterations [S1].

**Edge Cases and Pitfalls:**
*   **Local Optima:** Naive joint optimization on voxel-based NeRFs easily leads to suboptimal solutions due to the lack of spectral bias, which causes the model to overemphasize sharp edges [S1]. The proposed Gaussian filtering and coarse-to-fine schedule are critical to avoid this [S1].
*   **Precision:** The efficiency relies on the decomposition property; if the rank is too high, the computational overhead of the "efficient" convolution may approach that of brute-force 3D convolution [G].

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
