# Loss-based Adaptive Sampling

Also called: Grid-based Loss Sampling

## Overview
Loss-based Adaptive Sampling is a ray sampling strategy that allocates more samples to image regions with higher rendering loss, identified by partitioning the image into an 8x8 grid and computing mean loss per region [S1]. It differs from sibling methods like [Adaptive Sampling](1385-adaptive-sampling.md) by explicitly using a composite loss metric (gradient and pixel error) to drive a probability distribution for resampling, rather than relying solely on geometric depth or simple gradient magnitude [S1].

## How it works
**Inputs:**
*   Current neural radiance field model parameters.
*   Input source views (images).
*   Total sample count per image $n_i$.
*   Grid dimensions: $8 \times 8$ (64 regions).

**Outputs:**
*   A probability distribution $f_i[j]$ over the 64 regions.
*   A set of resampled pixels for the next training iteration.

**Data Structures:**
*   **Region Loss Array:** $\mathcal{H}_i$ of size 64, storing the mean loss for each region $R_j$.
*   **Probability Distribution:** $f_i$ of size 64, storing the normalized sampling probability for each region.
*   **Sample Set:** $s_i$, a list of pixel coordinates $(u,v)$ uniformly sampled from the input views.

**Steps:**
1.  **Uniform Sampling:** Randomly sample a uniform distribution of pixels $s_i$ from the input source views.
2.  **Partitioning:** Partition the image into a grid of 64 regions $R_j$ ($j=1..64$).
3.  **Loss Computation:** For each region $R_j$, compute the mean loss $\mathcal{H}_i[j]$ using the pixels $r_j = s_i \cap R_j$ uniformly sampled from that region. The loss combines gradient loss $e_i^g$ and pixel loss $e_i^p$:
    $$ \mathcal{H}_i[j] = \frac{1}{|r_j|} \sum_{(u,v) \in r_j} e_i^g[u, v] + e_i^p[u, v] $$
    where $|r_j|$ is the number of sampled pixels in region $j$ [S1].
4.  **Normalization:** Normalize the region losses into a probability distribution $f_i[j]$:
    $$ f_i[j] = \frac{\mathcal{H}_i[j]}{\sum_{m=1}^{64} \mathcal{H}_i[m]} $$
    [S1].
5.  **Resampling:** Allocate $n_i \cdot f_i[j]$ samples to each region $R_j$ for the next iteration, where $n_i$ is the total sample count per image. This allocates more samples to areas exhibiting elevated loss [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def loss_based_adaptive_sampling(model, images, n_i, grid_size=8):
    """
    model: Neural radiance field model
    images: List of input source views
    n_i: Total sample count per image
    grid_size: Dimension of the grid (default 8)
    """
    # Step 1: Uniform sampling
    # Randomly sample a uniform distribution of pixels s_i from the input source views
    s_i = uniform_sample_pixels(images, count=n_i)
    
    # Step 2: Partitioning
    # Partition the image into a grid of 64 regions
    regions = partition_image(images[0].shape, grid_size) # 8x8 grid
    
    # Initialize loss array
    region_losses = [0.0] * (grid_size * grid_size)
    region_counts = [0] * (grid_size * grid_size)
    
    # Step 3: Loss Computation
    for pixel in s_i:
        u, v = pixel
        region_idx = get_region_index(u, v, grid_size)
        
        # Compute gradient and pixel loss for this pixel
        # Note: e_i^g and e_i^p are specific to the model's loss function
        e_g = compute_gradient_loss(model, pixel)
        e_p = compute_pixel_loss(model, pixel)
        
        region_losses[region_idx] += e_g + e_p
        region_counts[region_idx] += 1
        
    # Average loss per region
    for j in range(len(region_losses)):
        if region_counts[j] > 0:
            region_losses[j] /= region_counts[j]
        else:
            region_losses[j] = 0.0 # Avoid division by zero, though unlikely with uniform sampling
            
    # Step 4: Normalization
    total_loss = sum(region_losses)
    if total_loss == 0:
        # Fallback to uniform distribution if all losses are zero
        probabilities = [1.0 / len(region_losses)] * len(region_losses)
    else:
        probabilities = [loss / total_loss for loss in region_losses]
        
    # Step 5: Resampling
    # Allocate n_i * f_i[j] samples to each region
    resampled_pixels = []
    for j, prob in enumerate(probabilities):
        num_samples = int(round(n_i * prob))
        # Sample uniformly from region j
        region_pixels = sample_from_region(j, grid_size, count=num_samples)
        resampled_pixels.extend(region_pixels)
        
    return resampled_pixels
```

**Parameters:**
*   **Grid size:** $8 \times 8$, resulting in 64 regions ($j=1..64$) [S1].
*   **$n_i$:** Total sample count per image used for resampling allocation [S1].

**Edge Cases and Pitfalls:**
*   **Empty Regions:** If a region $R_j$ has no sampled pixels ($|r_j|=0$), the loss is undefined. The pseudocode handles this by setting the loss to 0.0, which may result in zero probability for that region in the next iteration. [G]
*   **Zero Total Loss:** If all region losses are zero, the normalization step would result in division by zero. The pseudocode handles this by falling back to a uniform distribution. [G]
*   **Precision:** The loss values $e_i^g$ and $e_i^p$ should be computed with sufficient precision to avoid numerical instability in the normalization step. [G]

## Subtopics

### [Patch-based Ray Sampling](11953-patch-based-ray-sampling.md)
A ray sampling strategy where rays are sampled in contiguous 2D patches to form a local image grid, ensuring the spatial adjacency required for local structure analysis such as Discrete Wavelet Transform (DWT) or blur convolution. It is distinguished from random ray sampling, which captures global structures, by focusing on local pixel interference and high-frequency details.

## Operating characteristics
*   **Time Complexity:** The method requires computing loss for all sampled pixels, which is $O(n_i)$. The normalization and resampling steps are $O(64)$, which is constant. Thus, the overall complexity is dominated by the loss computation, $O(n_i)$.
*   **Memory Complexity:** The method requires storing the region losses and probabilities, which is $O(64)$, constant space. The sampled pixels require $O(n_i)$ space.
*   **Performance:** The method improves training convergence by focusing on regions with high error, particularly those with drastic color changes or depth variations [S1].
*   **Strengths:** Adapts to local scene complexity, improving efficiency in regions with high-frequency details.
*   **Limitations:** Requires computing both gradient and pixel loss, which may be computationally expensive. The fixed grid size may not adapt well to scenes with varying scales of detail.

## Background
Loss-based Adaptive Sampling was introduced in the context of Neural Implicit Scalable Encoding for SLAM (Nice-SLAM) [S1]. The method was developed to address the slow fitting efficiency in image regions with drastic color changes, which aligns with the rendering characteristics of NeRF [S1]. The approach is analogous to sample mining in conventional deep learning, aiming to identify pixels with inadequate convergence via the loss function and confer greater priority [S1].

## Sources
[S1] Efficient ray sampling for radiance fields reconstruction, https://doi.org/10.1016/j.cag.2023.11.005, fetched 2026-10-27
