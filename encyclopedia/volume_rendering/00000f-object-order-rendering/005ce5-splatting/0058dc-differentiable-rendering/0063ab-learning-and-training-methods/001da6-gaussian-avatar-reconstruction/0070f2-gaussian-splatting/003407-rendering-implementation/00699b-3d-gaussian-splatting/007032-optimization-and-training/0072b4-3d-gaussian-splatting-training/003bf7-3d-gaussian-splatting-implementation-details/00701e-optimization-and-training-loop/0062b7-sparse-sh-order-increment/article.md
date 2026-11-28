# Sparse SH Order Increment

Also called: SOI, Sparse Order Increment of SH

## Overview
Sparse SH Order Increment (SOI) is a memory-efficient training strategy for 3D Gaussian Splatting that dynamically increases the Spherical Harmonics (SH) order for a subset of Gaussians based on their view-dependent color error [S1]. Unlike standard 3DGS which assigns full 3rd-order SH coefficients to all primitives, SOI initializes all Gaussians with SH order 0 and selectively promotes only those with significant rendering errors to higher orders [S1]. This approach significantly reduces GPU memory consumption and storage overhead while maintaining rendering quality comparable to full-order SH, making it suitable for large-scale scenes [S1].

## How it works
**Inputs and Outputs**
*   **Input:** The current set of 3D Gaussians $\mathcal{G}$, the rendered images $\hat{I}$, ground-truth images $I_{gt}$, and the sparse rate parameter $r_s$.
*   **Output:** Updated Gaussian parameters where the SH order of a subset of Gaussians is incremented by one.

**Data Structures**
Each Gaussian $g_k$ stores its SH coefficients. Initially, only the 0th-order coefficient (constant color) is allocated. When the order is incremented, additional coefficients for higher degrees ($l=1, 2, \dots$) are allocated. The exact memory layout of SH coefficients follows the standard 3DGS convention (3 coefficients per degree for $l>0$), but the number of allocated coefficients per Gaussian varies dynamically [S1].

**Parameters**
*   $r_s$: Sparse rate, the percentage of Gaussians selected for order increment. Typical value: $0.2$ (20%) [S1].
*   Iteration steps: SOI is executed a fixed number of times (typically 3) during training [S1].
    *   For small scenes (30,000 total iterations): Steps at 16,000, 17,000, and 18,000 [S1].
    *   For large scenes (60,000 total iterations): Steps are scaled proportionately [S1].

**Algorithm Steps**
1.  **Initialization:** At the start of training, set the SH order to 0 for all Gaussians. This means only the 0th-order SH coefficient (view-independent color) is stored [S1].
2.  **Pruning:** Before executing SOI, perform Gaussian pruning to remove redundant primitives. This minimizes GPU memory overhead during the subsequent order increment [S1].
3.  **Error Calculation:** For each Gaussian $g_k$, calculate the view-based color difference $d_k$ across all training views. The formula is:
    $$ d_k = \frac{1}{N \cdot H \cdot W} \sum_{p=1}^{N} \mathbb{1}(k \in \mathcal{N}_p) \cdot w_k^p \cdot |C_p - C_p^{gt}| $$
    Where:
    *   $N$ is the number of training images.
    *   $H, W$ are the image height and width.
    *   $p$ iterates over all pixels in the $N$ images.
    *   $\mathbb{1}(k \in \mathcal{N}_p)$ is an indicator function equal to 1 if Gaussian $k$ overlaps pixel $p$, and 0 otherwise.
    *   $w_k^p$ is the contribution weight of Gaussian $k$ to pixel $p$.
    *   $C_p$ is the rendered color at pixel $p$.
    *   $C_p^{gt}$ is the ground truth color at pixel $p$.
    *   $| \cdot |$ denotes the absolute difference (L1 norm) [S1].
4.  **Selection:** Sort all Gaussians by their calculated difference $d_k$ in descending order. Select the top $r_s$ percentage of Gaussians from this sorted list [S1].
5.  **Increment:** Increment the SH order by one for the selected Gaussians. This involves allocating and initializing the next degree of SH coefficients [S1].
6.  **Repetition:** Repeat steps 2-5 every several iterations for a total of 3 times during the training process [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sparse_sh_order_increment(gaussians, rendered_images, gt_images, 
                              sparse_rate, current_iteration, total_iterations):
    """
    Perform Sparse SH Order Increment if the current iteration matches 
    the predefined schedule.
    
    Args:
        gaussians: List of Gaussian objects, each with an 'sh_order' attribute.
        rendered_images: Rendered images for all training views.
        gt_images: Ground truth images for all training views.
        sparse_rate: Fraction of Gaussians to promote (e.g., 0.2).
        current_iteration: Current training iteration number.
        total_iterations: Total number of training iterations.
        
    Returns:
        Updated list of Gaussians.
    """
    
    # Determine if SOI should be executed at this iteration
    # For small scenes (30k iters): 16000, 17000, 18000
    # For large scenes (60k iters): Proportionally scaled
    if not should_execute_soi(current_iteration, total_iterations):
        return gaussians

    # Step 1: Pruning to reduce memory overhead before incrementing
    gaussians = gaussian_pruning(gaussians)

    # Step 2: Calculate color difference for each Gaussian
    differences = []
    for k, g in enumerate(gaussians):
        d_k = 0.0
        count = 0
        # Iterate over all training views
        for view_idx in range(len(rendered_images)):
            rendered_img = rendered_images[view_idx]
            gt_img = gt_images[view_idx]
            H, W = rendered_img.shape[0], rendered_img.shape[1]
            
            # Get pixels overlapped by Gaussian k
            # Note: In implementation, this requires access to the rasterization 
            # data or re-rendering to get weights and overlap.
            # Assuming we have access to per-pixel contributions for g_k:
            for p in range(H * W):
                # Indicator function: 1 if g_k overlaps pixel p
                if gaussian_overlaps_pixel(g, p):
                    # Weight of g_k at pixel p
                    w_k_p = get_gaussian_weight(g, p)
                    # Color difference
                    color_diff = abs(rendered_img[p] - gt_img[p])
                    d_k += w_k_p * color_diff
                    count += 1
        
        # Normalize by N * H * W as per formula
        # Note: The sum in the formula is over all pixels in all N images.
        # The denominator is N * H * W.
        # If count < N*H*W, the un-overlapped pixels contribute 0 to the sum.
        n_images = len(rendered_images)
        d_k = d_k / (n_images * H * W)
        differences.append((d_k, k))

    # Step 3: Sort by difference descending
    differences.sort(key=lambda x: x[0], reverse=True)

    # Step 4: Select top r_s percentage
    num_to_select = int(len(gaussians) * sparse_rate)
    selected_indices = [item[1] for item in differences[:num_to_select]]

    # Step 5: Increment SH order for selected Gaussians
    for idx in selected_indices:
        gaussians[idx].sh_order += 1
        # Initialize new SH coefficients for the new order
        initialize_sh_coefficients(gaussians[idx], gaussians[idx].sh_order)

    return gaussians

def should_execute_soi(current_iteration, total_iterations):
    """
    Determine if SOI should run based on iteration schedule.
    """
    if total_iterations <= 30000:
        return current_iteration in [16000, 17000, 18000]
    else:
        # Scale proportionately for large scenes (60k iters)
        # 16000/30000 ~ 0.533, 17000/30000 ~ 0.566, 18000/30000 ~ 0.6
        # Approximate scaling:
        target_iters = [int(32000), int(34000), int(36000)] 
        # Note: Exact scaling factors not specified, but proportional logic is stated.
        # Implementation should use the specific scaled values from the source.
        return current_iteration in target_iters
```

**Edge Cases and Pitfalls**
*   **Memory Peaks:** Although SOI reduces overall memory, the increment operation itself can cause temporary memory spikes. Pruning before incrementing is critical to mitigate this [S1].
*   **Degenerate Input:** If a Gaussian has zero contribution to any pixel (e.g., completely occluded or out of view), its color difference $d_k$ will be 0. It will not be selected for increment, which is correct behavior [S1].
*   **Precision:** The color difference calculation involves summing over all pixels. For very large images, this sum can be large. Using floating-point precision consistent with the rendering pipeline is recommended [S1].

## Operating characteristics
**Time and Memory Complexity**
*   **Memory:** SOI significantly reduces storage and GPU memory overhead compared to full 3rd-order SH. For example, on Mip-NeRF360, storage drops from 742MB (vanilla 3DGS) to 98MB with $r_s=0.2$ [S1]. Peak GPU memory is also reduced (e.g., from 12.8GB to 8.8GB in ablation studies) [S1].
*   **Time:** Training time is slightly reduced or comparable to vanilla 3DGS. For Mip-NeRF360, training time decreased from 25m57s (vanilla) to 19m41s with SOI ($r_s=0.2$) [S1]. The overhead of calculating color differences and sorting Gaussians is offset by the reduced memory bandwidth and faster rendering due to fewer parameters [S1].

**Measured Performance**
On Mip-NeRF360 dataset:
*   $r_s=0.1$: SSIM 0.815, PSNR 27.27, LPIPS 0.218, FPS 215, Storage 78MB [S1].
*   $r_s=0.2$: SSIM 0.817, PSNR 27.38, LPIPS 0.216, FPS 218, Storage 98MB [S1].
*   $r_s=0.5$: SSIM 0.819, PSNR 27.46, LPIPS 0.213, FPS 220, Storage 158MB [S1].
*   Vanilla 3DGS (for reference): SSIM 0.821, PSNR 27.48, LPIPS 0.209, FPS 229, Storage 259MB [S1].

**Strengths and Limitations**
*   **Strengths:** Drastically reduces storage and GPU memory usage, enabling training of larger scenes. Maintains high rendering quality (SSIM/PSNR close to vanilla 3DGS). Improves rendering FPS due to fewer parameters to blend [S1].
*   **Limitations:** Requires careful scheduling of the increment steps. The optimal timing may vary with scene scale [S1]. The method assumes that view-dependent color errors are a good proxy for the need for higher SH orders, which is generally true but may not capture all geometric nuances [S1].

**When to Use**
Use SOI when GPU memory is constrained or when training large-scale scenes with high-resolution images. It is particularly effective for outdoor scenes where ambient light is low frequency, making high-order SH coefficients redundant for many Gaussians [S1].

## Background
Sparse SH Order Increment was introduced in the paper "EfficientGS: Streamlining Gaussian Splatting for Large-Scale High-Resolution Scene Representation" [S1]. The method addresses the high memory consumption of 3D Gaussian Splatting, which typically uses 3rd-order SH for all Gaussians. The authors observed that many Gaussians, especially in outdoor scenes, do not require high-order SH to represent their view-dependent color accurately. By starting with SH order 0 and selectively incrementing based on color error, EfficientGS achieves a more efficient representation [S1].

## Sources
[S1] EfficientGS: Streamlining Gaussian Splatting for Large-Scale High-Resolution Scene Representation, https://doi.org/10.1109/mmul.2025.3543224, fetched 2026-10-30
