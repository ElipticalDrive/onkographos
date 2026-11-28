# Simple-ZipNeRF

Also called: Augmented ZipNeRF, Simple-RF ZipNeRF

## Overview
Simple-ZipNeRF is a regularized variant of ZipNeRF designed to suppress floaters and improve geometry reconstruction in sparse input scenarios [S1]. It operates by training an auxiliary "augmented" model with reduced hash table capacity alongside the main model, using the augmented model's depth estimates to supervise the main model [S1]. Unlike the parent method which applies general anti-aliasing and regularization principles, Simple-ZipNeRF specifically targets the overfitting caused by large hash tables in sparse settings by forcing feature sharing through capacity reduction [S1].

## How it works
Simple-ZipNeRF trains two models simultaneously: a main ZipNeRF model and an augmented ZipNeRF model [S1]. The augmented model serves as a regularizer by providing depth priors that guide the main model toward simpler, more geometrically consistent solutions [S1].

**Inputs and Outputs:**
*   **Inputs:** Sparse set of input images (typically 4–36 views), camera poses, and intrinsics [S1].
*   **Outputs:** A trained main ZipNeRF model capable of rendering novel views with reduced floaters [S1].

**Data Structures:**
The architecture mirrors ZipNeRF, utilizing multi-resolution hash grids for spatial encoding [S1]. The key difference lies in the hash table size of the augmented model:
*   **Main Model:** Uses the standard ZipNeRF hash table size $T = 2^{21}$ [S1].
*   **Augmented Model:** Uses a reduced hash table size $T_s = 2^{11}$ [S1].
*   **MLPs:** The augmented model uses the same MLP architecture and hyperparameters as the main model, differing only in hash table size [S1].

**Training Procedure:**
The training loop minimizes a combined loss function. Let $L_m$ be the photometric loss for the main model, $L_a$ be the photometric loss for the augmented model, and $L_{aug}$ be the depth regularization loss [S1].

1.  **Initialize** the main ZipNeRF model with standard parameters and the augmented ZipNeRF model with $T_s = 2^{11}$ [S1].
2.  **Iterate** for $K$ iterations (typically 50k+):
    a.  **Sample Rays:** For each pixel, sample a set of points along the ray using the main model's proposal network [S1].
    b.  **Query Main Model:** Compute color and density for the sampled points using the main hash grid and MLP [S1].
    c.  **Query Augmented Model:** Compute color and density for the same points using the reduced hash grid and MLP [S1].
    d.  **Compute Depth:** Estimate depth from the augmented model's volume rendering [S1].
    e.  **Validate Depth:** Check if the depth is reliable by reprojecting the 3D point to the closest training view and verifying consistency [S1]. If the depth is unreliable, discard it [S1].
    f.  **Compute Losses:**
        *   $L_m = \| C_m - C_{gt} \|^2$ [S1].
        *   $L_a = \| C_a - C_{gt} \|^2$ [S1].
        *   $L_{aug}$ is applied only when the depth estimates are validated as accurate [S1]. The specific form of $L_{aug}$ is not provided in the sources [S1].
        *   Note: Sparse depth loss $L_{sd}$ is **not** used, as COLMAP often fails or provides noisy sparse depth in these scenarios [S1].
    g.  **Total Loss:** $L_{total} = \lambda_m L_m + \lambda_a L_a + \lambda_{aug} L_{aug}$ [S1].
    h.  **Update:** Perform gradient descent on both models [S1].

**Parameters:**
*   **Hash Table Size (Augmented):** $T_s = 2^{11}$ [S1].
*   **Hash Table Size (Main):** $T = 2^{21}$ [S1].
*   **Loss Weights:** $\lambda_m = 1, \lambda_a = 1, \lambda_{aug} = 10$ [S1].
*   **Other Losses:** $\lambda_{sd} = 0, \lambda_{cfc} = 0, \lambda_{mc} = 0$ [S1].
*   **Regularization Start:** $L_{aug}$ is imposed after 5k iterations [S1].
*   **Hyperparameters:** $k=5, e_\tau=0.2$ (used in the regularization mechanism, exact definition not provided in sources) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data structures and helpers
class HashGridNetwork:
    def __init__(self, hash_table_size):
        self.hash_table_size = hash_table_size
        self.mlp = ... # Standard MLP for decoding features
    def forward(self, rays):
        # Query hash grid, trilinear interpolate, decode with MLP
        return color, density

def sample_rays(batch_size):
    # Standard ray sampling for a batch of pixels
    return rays

def volume_render_depth(density):
    # Standard volume rendering to compute expected depth
    return depth

def reproject_and_check_consistency(depth, rays, training_views):
    # Reproject 3D points along rays to closest training view and check color consistency
    return valid_mask

def depth_regularization_loss(depth, main_net, valid_mask):
    # Use accurate depth estimates to supervise main model geometry
    # Exact formula not provided in sources [S1]
    return loss

# Initialization
main_net = HashGridNetwork(hash_table_size=2**21)
aug_net = HashGridNetwork(hash_table_size=2**11)
optimizer = Adam([main_net.parameters(), aug_net.parameters()])

# Training Loop
for iteration in range(50000):
    rays = sample_rays(batch_size)
    gt_colors = get_ground_truth(rays)

    # Forward pass Main Model
    c_m, d_m = main_net.forward(rays)
    loss_m = mean_squared_error(c_m, gt_colors)

    # Forward pass Augmented Model
    c_a, d_a = aug_net.forward(rays)
    loss_a = mean_squared_error(c_a, gt_colors)

    # Compute Depth from Augmented Model
    depth_a = volume_render_depth(d_a)

    # Validate Depth Reliability
    valid_mask = reproject_and_check_consistency(depth_a, rays, training_views)

    # Compute Regularization Loss (only on valid depths, after 5k iterations)
    if iteration > 5000:
        loss_aug_reg = depth_regularization_loss(depth_a, main_net, valid_mask)
    else:
        loss_aug_reg = 0.0

    # Total Loss
    lambda_m = 1.0
    lambda_a = 1.0
    lambda_aug = 10.0
    total_loss = (lambda_m * loss_m) + (lambda_a * loss_a) + (lambda_aug * loss_aug_reg)

    # Backpropagate and Update
    optimizer.zero_grad()
    total_loss.backward()
    optimizer.step()
```

**Edge Cases and Pitfalls:**
*   **Blur:** The reduced hash table size introduces blur in the augmented model, which may not be reflected in quantitative metrics like PSNR despite visual improvements [S1].
*   **Sparse Depth:** Sparse depth loss is ineffective because COLMAP often fails or provides noisy depth in sparse settings [S1].
*   **Performance Limits:** The augmentation does not help if the base model has very few input images (performance too low) or a large number of input views (performance saturated) [S1].
*   **Persistent Floaters:** Some translucent floaters persist, particularly in regions visible in only one training view where reliable depth supervision is unavailable [S1].

## Operating characteristics
*   **Training Time:** Simple-ZipNeRF takes approximately twice the training time of standard ZipNeRF because it employs a single augmentation with exactly the same number of queries as the main model [S1]. On an NVIDIA RTX 2080 Ti 11GB GPU, ZipNeRF takes ~2.0 hours, while Simple-ZipNeRF takes ~4.2 hours [S1].
*   **Inference Time:** At inference time, Simple-ZipNeRF takes exactly the same time and memory as ZipNeRF because the augmentations are disabled [S1]. Inference time is ~13 seconds per frame [S1].
*   **Memory:** Training memory is higher due to the second model. ZipNeRF uses ~6.7 GB, while Simple-ZipNeRF uses ~8.6 GB [S1]. Inference memory is identical (~4.8 GB) [S1].
*   **Performance:**
    *   **MipNeRF360 (12 views):** PSNR 17.60, SSIM 0.5245, LPIPS 0.4876 [S1].
    *   **MipNeRF360 (20 views):** PSNR 21.03, SSIM 0.6456, LPIPS 0.3421 [S1].
    *   **MipNeRF360 (36 views):** PSNR 24.19, SSIM 0.7458, LPIPS 0.2390 [S1].
    *   **NeRF-Synthetic (4 views):** PSNR 11.50, SSIM 0.7715, LPIPS 0.3878 [S1].
    *   **NeRF-Synthetic (8 views):** PSNR 15.88, SSIM 0.8063, LPIPS 0.2461 [S1].
    *   **NeRF-Synthetic (12 views):** PSNR 20.51, SSIM 0.8531, LPIPS 0.1532 [S1].
*   **Strengths:** Significantly reduces floaters and improves geometry in sparse input scenarios [S1].
*   **Limitations:** Increased training time and memory; ineffective when input views are extremely sparse or extremely dense; may introduce blur [S1].

## Background
Simple-ZipNeRF is part of the "Simple Radiance Fields" (Simple-RF) family of models, which regularize radiance fields to choose simpler solutions over complex ones [S1]. The method was proposed to address the issue of floaters caused by large hash tables in ZipNeRF when trained with sparse inputs [S1]. The approach was evaluated on forward-facing scenes (NeRF-LLFF), unbounded forward-facing scenes (RealEstate-10K), unbounded 360° scenes (MipNeRF360), and bounded 360° scenes (NeRF-Synthetic) [S1].

## Sources
[S1] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
