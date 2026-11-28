# Load Balancing for Pixel-parallel Splatting

Also called: Load balancing algorithm, Load balancing loss

## Overview
A strategy to address uneven computational load across pixel threads in Gaussian rasterization by reducing the number of Gaussians rendered for heavy-load pixels and increasing it for light-load pixels [S1]. It sets itself apart from sibling methods like [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md) by explicitly optimizing the *count* of splats per pixel to balance GPU thread utilization, rather than just spatially partitioning the image [S1].

## How it works
**Inputs:**
- Set of $N$ 3D Gaussians.
- Camera parameters (view/projection matrices).
- Image resolution (width $W$, height $H$).
- Threshold for opacity contribution (implicit in the indicator function).

**Outputs:**
- Modified set of Gaussians to be rendered per pixel, or a loss value guiding training to achieve this distribution.

**Data Structures:**
- `Gaussian`: Standard 3DGS representation (mean, covariance, opacity, SH coefficients).
- `PixelLoad`: Integer count of Gaussians contributing to a specific pixel.

**Steps:**
1. **Compute Pixel Loads:** For each pixel $i$ in the image, determine the number of Gaussians $\ell_i$ that contribute to it. A Gaussian $j$ contributes to pixel $i$ if it projects within the pixel's bounds and its opacity is significant.
   $$ \ell_i = \sum_{j \in N} \mathbb{1}_{G_j, p_i} $$
   where $\mathbb{1}_{G_j, p_i}$ is an indicator function equal to 1 if Gaussian $j$ contributes to pixel $i$, and 0 otherwise [S1].
2. **Identify Load Imbalance:** Calculate the standard deviation of $\ell_i$ across all pixels. High standard deviation indicates uneven load, causing GPU thread waiting [S1].
3. **Apply Load Balancing Strategy:**
   - For **heavy-load pixels** (where $\ell_i$ is high), reduce the number of Gaussians rendered. This can be done by culling Gaussians with low opacity or small contribution to the pixel's color/transmittance.
   - For **light-load pixels** (where $\ell_i$ is low), increase the number of Gaussians rendered. This may involve relaxing culling thresholds or including Gaussians that were previously discarded.
4. **Training Integration (Load Balancing Loss):** During training, a loss term $\lambda_{load}$ is added to the optimization objective to encourage this balanced distribution. The weight $\lambda_{load}$ is typically set to 0.45, while the sum of weights for remaining losses (e.g., L1, SSIM) is reduced to 0.55 [S1].

**Parameters:**
- $\lambda_{load}$: Weight for the load balancing loss term. Typical value: 0.45 [S1].
- $\lambda_{remaining}$: Sum of weights for other loss terms (L1, SSIM). Typical value: 0.55 [S1].
- Opacity threshold: Used to determine if a Gaussian contributes to a pixel. Higher values accelerate rendering but may reduce quality [S1].

**Edge Cases and Pitfalls:**
- **Quality Degradation:** Aggressively reducing Gaussians for heavy-load pixels can lead to visible artifacts or loss of detail [S1].
- **Overhead:** Computing exact pixel loads for every Gaussian can be expensive. Approximations or hierarchical methods may be needed for large scenes.
- **Thread Divergence:** If load balancing is not effective, GPU threads may still diverge, reducing efficiency [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_pixel_loads(gaussians, image_width, image_height):
    """
    Computes the number of Gaussians contributing to each pixel.
    Returns a 2D array of loads.
    """
    loads = np.zeros((image_height, image_width), dtype=np.int32)
    for g in gaussians:
        # Project Gaussian center to screen space
        center_2d = project_to_screen(g.mean, g.view_matrix, g.proj_matrix)
        # Determine bounding box of Gaussian in screen space
        bbox = get_gaussian_bbox(center_2d, g.covariance)
        # Iterate over pixels in bounding box
        for y in range(bbox.y_min, bbox.y_max):
            for x in range(bbox.x_min, bbox.x_max):
                if is_within_pixel_bounds(center_2d, g.covariance, x, y):
                    # Check if opacity contribution is significant
                    if get_opacity_at_pixel(g, x, y) > THRESHOLD:
                        loads[y, x] += 1
    return loads

def apply_load_balancing(gaussians, loads, lambda_load, lambda_remaining):
    """
    Adjusts Gaussian rendering based on pixel loads.
    Returns modified Gaussians or loss value.
    """
    # Calculate mean and standard deviation of loads
    mean_load = np.mean(loads)
    std_load = np.std(loads)
    
    # Define heavy and light load thresholds
    heavy_threshold = mean_load + std_load
    light_threshold = mean_load - std_load
    
    # Identify heavy and light pixels
    heavy_pixels = np.where(loads > heavy_threshold)
    light_pixels = np.where(loads < light_threshold)
    
    # For heavy pixels, reduce Gaussian count (e.g., by increasing opacity threshold)
    # For light pixels, increase Gaussian count (e.g., by decreasing opacity threshold)
    # This part is typically handled during training via the loss function
    
    # Load Balancing Loss
    # Encourage uniform load distribution
    load_loss = std_load
    
    # Total Loss
    total_loss = lambda_load * load_loss + lambda_remaining * other_losses
    return total_loss
```

## Subtopics

### [Gradient-Constrained Load Balancing Loss](21475-gradient-constrained-load-balancing-loss.md)
Gradient-Constrained Load Balancing Loss (LGC-load) is a regularization term used during the training of 3D Gaussian Splatting models to optimize computational efficiency by balancing the rendering workload across parallel pixel threads. It addresses the problem of uneven Gaussian distribution, where complex scene regions require more Gaussians for fidelity while simple regions do not, causing thread idle time in pixel-parallel rasterization. Unlike rigid load balancing that enforces uniform Gaussian counts, LGC-load adapts the allowed variance in Gaussian density per pixel based on local image gradients, thereby preserving reconstruction quality in intricate areas while reducing redundancy in simple areas. It builds on [Load Balancing for Pixel-parallel Splatting](8373-load-balancing-for-pixel-parallel-splatting.md).

### Load Distribution Unit (LDU) Workload Balancing
A workload distribution strategy to address severe imbalance in tile workloads, using early stopping depth prediction and Morton-order traversal to balance load across rasterization blocks [S2]. It involves preprocessing by discarding Gaussians beyond predicted early stopping depth, computing an ideal average workload $W$, and assigning tiles to blocks sequentially, deferring tiles if the cumulative Gaussian-tile pairs exceed $(1 + 1/N)W$. It employs Morton-order traversal to group spatially adjacent tiles, reducing memory access overhead, and sorts tiles within each block from light to heavy to prevent rasterization bubbles [S2].

### LS-Gaussian
A method that reuses image regions and schedules remaining tiles from predicted load [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the number of Gaussians $N$ and image resolution $W \times H$. Computing pixel loads is $O(N \cdot A)$ where $A$ is the average area of a Gaussian in pixels. Load balancing adjustments are $O(W \cdot H)$.
- **Memory Complexity:** $O(W \cdot H)$ for storing pixel loads.
- **Performance:** Improves rendering efficiency by reducing GPU thread waiting time due to load imbalance [S1]. Can lead to significant speedups in scenes with high variance in Gaussian density.
- **Strengths:** Simple to implement, directly addresses GPU thread divergence.
- **Limitations:** May reduce rendering quality if not carefully tuned. Requires accurate estimation of pixel loads.
- **When to Use:** In scenes with highly uneven Gaussian distribution, where some pixels are much more complex than others.
- **When Not to Use:** In scenes with uniform Gaussian distribution, where load balancing provides little benefit.

## Background
Originated from the need to improve the efficiency of 3D Gaussian Splatting rendering on GPUs, where thread divergence due to uneven workload is a major bottleneck [S1]. The method was proposed to balance the number of Gaussians rendered per pixel, ensuring that all GPU threads finish their work at approximately the same time [S1].

## Sources
[S1] AdR-Gaussian: Accelerating Gaussian Splatting with Adaptive Radius, https://doi.org/10.1145/3680528.3687675, fetched 2026-10-26
[S2] No Redundancy, No Stall: Lightweight Streaming 3D Gaussian Splatting for Real-time Rendering, https://doi.org/10.1109/iccad66269.2025.11240950, fetched 2026-10-30
[S3] From Splats to Silicon: Rethinking Computational Efficiency of 3DGS, https://arxiv.org/abs/2609.06157, fetched 2026-10-29 (unofficial)
