# Sparse SH Order Increment

Also called: SOI, Sparse Order Increment of SH

## Overview
Sparse SH Order Increment (SOI) is a training optimization strategy for 3D Gaussian Splatting that reduces GPU memory consumption and storage overhead by initializing all Gaussians with Spherical Harmonic (SH) order 0 and selectively promoting them to higher orders based on view-dependent color error [S1]. Unlike standard 3D Gaussian Splatting which assigns full 3rd-order SH coefficients to every Gaussian, SOI exploits the redundancy in SH representations by only assigning higher-order coefficients to Gaussians that exhibit significant color deviation from ground truth across training views [S1]. This approach guarantees better rendering quality for a given sparse rate compared to random selection of Gaussians for order increment [S1].

## How it works
**Inputs:**
*   Set of 3D Gaussians $\mathcal{G}$, each defined by mean $\mu$, covariance $\Sigma$, opacity $\alpha$, and SH coefficients $c_{k,l,m}$ for order $l$ and degree $m$.
*   Training dataset consisting of $N$ images with dimensions $H \times W$.
*   Sparse rate parameter $r_s$ (fraction of Gaussians to promote).
*   Iteration schedule for SOI execution.

**Outputs:**
*   Updated set of Gaussians $\mathcal{G}'$ with potentially increased SH orders for a subset of Gaussians.

**Data Structures:**
*   **Gaussian Attributes:** Each Gaussian $g_k$ stores its current SH order $L_k$ (integer, typically $0 \le L_k \le 3$).
*   **Color Difference Metric:** A scalar value $d_k$ computed for each Gaussian $k$ to rank them.

**Numbered Steps:**

1.  **Initialization:** At the start of training, set the SH order $L_k = 0$ for all Gaussians $k \in \mathcal{G}$. This minimizes initial GPU memory usage [S1].
2.  **Pruning (Pre-condition):** Before the first SOI execution, perform Gaussian pruning to remove low-opacity or redundant Gaussians. This step is critical to minimize GPU memory overhead before allocating higher-order SH coefficients [S1].
3.  **Compute Color Difference:** For each Gaussian $k$, calculate the view-based color difference $d_k$ across all training views. The formula is:
    $$ d_k = \frac{1}{N \cdot H \cdot W} \sum_{p=1}^{N} \mathbb{1}(k \in \mathcal{N}_p) \cdot w_k^p \cdot |C_p - C_p^{gt}| $$
    Where:
    *   $N$ is the number of training images.
    *   $H, W$ are the image height and width.
    *   $p$ iterates over all training images (or pixels, depending on interpretation of the summation index in the source; the source formula sums over $p=1$ to $N$, implying image-level or aggregated pixel-level error, but the denominator $N \cdot H \cdot W$ suggests normalization by total pixel count. The source text says "sum_{p=1}^{N}" but defines $p$ as pixels in the variable list. However, the denominator $N \cdot H \cdot W$ implies the sum is over all pixels in all images. Let's look closely at the source formula: $d_k = \frac{1}{N \cdot H \cdot W} \sum_{p=1}^{N} ...$. The source text says "$p$ iterates over all pixels" but the sum limit is $N$. This is a contradiction in the source notation. However, the denominator $N \cdot H \cdot W$ strongly implies the sum is over all pixels in the dataset. The indicator $\mathbb{1}(k \in \mathcal{N}_p)$ checks if Gaussian $k$ contributes to pixel $p$. $w_k^p$ is the contribution weight. $C_p$ is rendered color, $C_p^{gt}$ is ground truth. We will assume the sum is over all pixels $p$ in the training set, normalized by total pixel count $N \cdot H \cdot W$ [S1].
    *   $\mathbb{1}(k \in \mathcal{N}_p)$ is an indicator function equal to 1 if Gaussian $k$ overlaps pixel $p$ (i.e., contributes to its color), and 0 otherwise.
    *   $w_k^p$ is the contribution weight of Gaussian $k$ to pixel $p$ during alpha blending.
    *   $C_p$ is the rendered color at pixel $p$.
    *   $C_p^{gt}$ is the ground truth color at pixel $p$.
    *   $| \cdot |$ denotes the absolute difference (or L1 norm per channel) [S1].
4.  **Ranking:** Sort all Gaussians in $\mathcal{G}$ in descending order based on their computed difference $d_k$.
5.  **Selection:** Select the top $r_s \cdot |\mathcal{G}|$ Gaussians from the sorted list. These are the Gaussians with the highest color deviation, indicating a higher demand for complex color representation [S1].
6.  **Increment:** For each selected Gaussian, increment its SH order by one ($L_k \leftarrow L_k + 1$). This involves allocating memory for the new SH coefficients and initializing them (typically to zero or inheriting from a lower-order approximation if applicable, though the source does not specify initialization of new coefficients, standard practice implies zero-initialization or copying lower-order terms) [S1].
7.  **Iteration:** Repeat steps 2-6 every several iterations. For small scenes (30,000 total iterations), SOI is executed at iterations 16,000, 17,000, and 18,000. For large scenes (60,000 total iterations), the steps are modified proportionately (e.g., 32,000, 34,000, 36,000) [S1]. The total number of SOI executions is 3 [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sparse_sh_order_increment(gaussians, training_data, rs, current_iteration, total_iterations):
    """
    Perform Sparse SH Order Increment if the current iteration matches the schedule.
    
    Args:
        gaussians: List of Gaussian objects, each with attribute 'sh_order' and 'id'.
        training_data: Dataset containing N images of size H x W.
        rs: Sparse rate (fraction of Gaussians to promote).
        current_iteration: Current training step.
        total_iterations: Total number of training steps.
        
    Returns:
        Updated list of Gaussians.
    """
    
    # Determine if SOI should be executed
    # For small scenes (30k), steps are 16000, 17000, 18000
    # For large scenes (60k), steps are proportional (approx 32000, 34000, 36000)
    # The source states SOI is performed 3 times.
    
    if current_iteration not in get_soI_schedule(total_iterations):
        return gaussians

    # Step 2: Pruning is performed before SOI. 
    # This is handled by the parent training loop's densification/pruning step.
    # We assume gaussians are already pruned.
    
    # Step 3: Compute color difference d_k for each Gaussian
    differences = []
    for g in gaussians:
        d_k = 0.0
        total_pixels = 0
        for image_idx in range(len(training_data)):
            image = training_data[image_idx]
            # Render the image using current Gaussians to get C_p
            # This requires the differentiable rasterizer from the parent loop
            rendered_image = render_image(gaussians, image.camera_pose)
            
            # Calculate contribution of Gaussian g to each pixel
            # Note: In practice, this might be accumulated during rendering
            # Here we assume we can access per-pixel contributions
            for y in range(image.height):
                for x in range(image.width):
                    # Check if Gaussian g contributes to pixel (x,y)
                    # weight_k^p is the alpha weight of g at pixel (x,y)
                    weight = get_contribution_weight(g, x, y, rendered_image)
                    
                    if weight > 0:
                        # Indicator function 1(k in N_p) is 1 if weight > 0
                        indicator = 1
                        color_diff = abs(rendered_image[y, x] - image.gt[y, x])
                        d_k += indicator * weight * color_diff
                        total_pixels += 1
        
        # Normalize by total pixel count N * H * W
        # Note: The source formula divides by N*H*W. 
        # If total_pixels is not equal to N*H*W (due to empty pixels), 
        # we still divide by the global total to match the source definition.
        global_total_pixels = len(training_data) * image.height * image.width
        d_k /= global_total_pixels
        differences.append((g.id, d_k))
    
    # Step 4: Sort by difference descending
    differences.sort(key=lambda x: x[1], reverse=True)
    
    # Step 5: Select top rs percentage
    num_to_promote = int(rs * len(gaussians))
    selected_ids = [diff[0] for diff in differences[:num_to_promote]]
    
    # Step 6: Increment SH order for selected Gaussians
    for g in gaussians:
        if g.id in selected_ids:
            if g.sh_order < 3: # Max order is typically 3
                g.sh_order += 1
                # Initialize new SH coefficients to zero or appropriate values
                initialize_sh_coefficients(g, g.sh_order)
                
    return gaussians

def get_soI_schedule(total_iterations):
    """
    Returns the iteration steps for SOI execution.
    Based on source: 3 times total.
    Small scenes (30k): 16000, 17000, 18000.
    Large scenes (60k): Proportional.
    """
    if total_iterations <= 30000:
        return [16000, 17000, 18000]
    else:
        # Proportional scaling
        # 16000/30000 = 0.5333
        # 17000/30000 = 0.5666
        # 18000/30000 = 0.6000
        scale = total_iterations / 30000.0
        return [int(16000 * scale), int(17000 * scale), int(18000 * scale)]

def initialize_sh_coefficients(gaussian, new_order):
    """
    Initialize SH coefficients for the new order.
    Source does not specify, but standard practice is zero-initialization.
    """
    # Allocate memory for SH coefficients of order new_order
    # Number of coefficients for order L is (L+1)^2
    num_coeffs = (new_order + 1) ** 2
    gaussian.sh_coeffs = [0.0] * num_coeffs
```

**Parameters:**
*   $r_s$ (Sparse Rate): The fraction of Gaussians selected to receive increased SH order. Typical value: 0.2 (20%) [S1]. Higher $r_s$ slightly improves quality; $r_s=0.5$ yields quality similar to full SH, proving SH redundancy [S1].
*   **Iteration Steps:** For small scenes (30,000 iterations), SOI is executed at iterations 16,000, 17,000, and 18,000 [S1]. For large scenes (60,000 iterations), steps are modified proportionately [S1].
*   **Total Executions:** 3 times during training [S1].

**Edge Cases and Pitfalls:**
*   **Memory Overflow:** If $r_s$ is too high or pruning is insufficient, GPU memory may exceed limits during the allocation of higher-order SH coefficients. Pruning must be performed before SOI [S1].
*   **Precision:** The color difference calculation involves summing over all pixels, which may require double precision to avoid numerical instability if the number of pixels is very large [G].
*   **Boundary Conditions:** If the number of Gaussians is small, $r_s \cdot |\mathcal{G}|$ may be zero. Ensure at least one Gaussian is promoted if $r_s > 0$ and $|\mathcal{G}| > 0$ [G].
*   **Max Order:** SH order cannot exceed 3 (standard for 3DGS). Ensure increments do not exceed this limit [G].

## Subtopics

### Random SH Order Increment
A baseline method where Gaussians are selected randomly to increment SH order. SOI outperforms this method in rendering quality for the same $r_s$ value [S1].

### Full SH Order
The standard 3DGS approach where all Gaussians use 3rd-order SH. SOI achieves comparable quality with significantly reduced storage and memory [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** SOI significantly reduces peak GPU memory consumption and final storage size compared to full SH. For example, on Mip-NeRF360, SOI with $r_s=0.2$ reduces storage to 98MB compared to higher values for larger $r_s$ or full SH (implied by "significantly reduce storage overhead") [S1]. Peak GPU memory is reduced from 12.8GB (baseline) to 8.8GB with SOI ($r_s=0.2$) [S1].
*   **Time:** Training time is slightly reduced or comparable. On Mip-NeRF360, SOI ($r_s=0.2$) training time is 19m41s, compared to 25m57s for the baseline with selective densification only [S1].

**Measured Performance:**
*   **Mip-NeRF360 (rs=0.1):** SSIM 0.815, PSNR 27.27, LPIPS 0.218, FPS 215, Storage 78MB [S1].
*   **Mip-NeRF360 (rs=0.2):** SSIM 0.817, PSNR 27.38, LPIPS 0.216, FPS 218, Storage 98MB [S1].
*   **Mip-NeRF360 (rs=0.5):** SSIM 0.819, PSNR 27.46, LPIPS 0.213, FPS 220, Storage 158MB [S1].
*   **Comparison:** SOI with $r_s=0.2$ achieves similar quality to full SH but with much lower storage and memory [S1].

**Strengths and Limitations:**
*   **Strengths:** Reduces GPU memory overhead, allowing training of larger scenes. Reduces storage size. Maintains high rendering quality. Faster training due to reduced memory pressure and potentially fewer operations per Gaussian in early stages.
*   **Limitations:** Requires careful tuning of $r_s$ and iteration steps. Performance may degrade if the wrong module execution time is chosen for different scene scales [S1].

**When to Use:**
*   Use when GPU memory is limited or storage size is a concern.
*   Use for large-scale scenes where full SH is prohibitive.
*   Use when training time needs to be reduced.

**When Not to Use:**
*   Use when maximum possible rendering quality is required regardless of memory/storage (use full SH).
*   Use when scene complexity is very low and SH order 0 is sufficient (no need for SOI).

## Background
**Origin:**
Sparse SH Order Increment was proposed in the paper "EfficientGS: Streamlining Gaussian Splatting for Large-Scale High-Resolution Scene Representation" [S1]. The paper was published in 2025 (based on the DOI date) and presented by authors from the EfficientGS project.

**History and Development:**
The method builds upon the original 3D Gaussian Splatting framework [Kerbl et al., 2023] and addresses the limitations of high memory consumption and storage size associated with full 3rd-order SH coefficients. It was developed to enable efficient training and rendering of large-scale, high-resolution scenes.

**Where it is used:**
Used in the EfficientGS framework for 3D Gaussian Splatting optimization. Validated on datasets including Mip-NeRF360, Tanks&Temples, and Deep Blending [S1].

## Sources
[S1] EfficientGS: Streamlining Gaussian Splatting for Large-Scale High-Resolution Scene Representation, https://doi.org/10.1109/mmul.2025.3543224, fetched 2026-10-30
