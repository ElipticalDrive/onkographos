# S3IM

Also called: Multiplex Loss, Multiplex Training via S3IM, S3IM (Stochastic Structural SIMilarity), S3IM Training, S3IM loss, Stochastic Structural SIMilarity (S3IM), Stochastic Structural SIMilarity Training

## Overview
S3IM (Stochastic Structural SIMilarity) is a multiplex loss function for neural fields that captures non-local structural similarity by processing multiple data points (pixels/rays) as a whole set, rather than individually [S1]. It addresses the limitation of point-wise losses (like MSE) which fail to use collective supervision of distant pixels [S1]. Unlike conventional MSE loss which is computed pixel-wisely, S3IM cannot be expressed as the sum of any other loss computed over individual pixels from the batch [S1]. It diverges from conventional point-to-point loss functions by analyzing various data points collectively to obtain non-local structural details [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A minibatch of rays $\mathcal{R}$ sampled from the dataset; ground-truth pixel colors $\mathcal{C} = \{\mathbf{C}(\mathbf{r}) | \mathbf{r} \in \mathcal{R}\}$; rendered pixel colors $\hat{\mathcal{C}} = \{\hat{\mathbf{C}}(\mathbf{r}) | \mathbf{r} \in \mathcal{R}\}$ computed via differentiable volume rendering [S1].
*   **Outputs:** The S3IM loss value $L_{\SSSIM}$ and the total multiplex loss $L_{\mathrm{M}}$ used for gradient backpropagation [S1].

**Data Structures:**
*   **Minibatch $\mathcal{R}$:** A set of $B$ rays. Typical batch size $B=1024$ [S1].
*   **Pixel Vectors:** $\mathbf{C}(\mathbf{r})$ and $\hat{\mathbf{C}}(\mathbf{r})$ are RGB color vectors in $[0, 1]$ [S1].
*   **Stochastic Patch:** A subset of pixels from the minibatch treated as a pseudo-image for SSIM computation.

**Parameters:**
*   $M$: Number of stochastic estimations (patches). Default $M=10$ to reduce stochasticity, though $M=1$ yields nearly identical results [S1].
*   $K$: Kernel size for SSIM computation. Default $K=4$ (i.e., $4 \times 4$ window) [S1].
*   $s$: Stride size for SSIM computation. Default $s=K$ [S1].
*   $\lambda$: Weight for the S3IM loss term. Requires fine-tuning [S1].
*   $C_1, C_2, C_3$: SSIM stability constants. $C_1 = (K_1 L)^2$, $C_2 = (K_2 L)^2$, $C_3 = C_2 / 2$. Typical values: $K_1 = 0.01$, $K_2 = 0.03$, $L = 1$ for RGB images [S1].

**Formulas:**
The total multiplex loss is defined as [S1]:
$$ L_{\mathrm {M}} (\Theta ) = \frac {1}{\|\mathcal {R}\|}\sum _{\bm {r}\in \mathcal {R}} l_{\MSE }(\Theta , \bm {r}) + \lambda L_{\SSSIM }(\Theta , \mathcal {R}) $$
where $l_{\MSE}$ is the pixel-wise MSE loss. For neural surface representations like NeuS, the color loss is replaced by the multiplex color loss using L1 form [S1]:
$$ L_{\mathrm {M},\mathrm {color}} = L_{\mathrm {color}} + \lambda L_{\SSSIM }, \quad L_{\mathrm {color}} = \frac{1}{\|\mathcal {R}\|}\sum _{\bm {r}\in \mathcal {R}} \| \hat {\bm {C}}(\bm {r}) - \bm {C}(\bm {r}) \|_{1} $$

The S3IM value is computed as the average of $M$ SSIM estimators over stochastic patches [S1]:
$$ \SSSIM (\hat {\mathcal {R}},\mathcal {R}) = \frac {1}{M} \sum _{m=1}^{M} \SSIM (\Patch ^{(m)}(\hat {\mathcal {C}}), \Patch ^{(m)}(\mathcal {C})) $$
The S3IM loss is defined as [S1]:
$$ L_{\SSSIM }(\Theta , \mathcal {R}) = 1 - \SSSIM (\hat {\mathcal {R}},\mathcal {R}) $$

SSIM is defined as the product of luminance, contrast, and structure terms [S1]:
$$ \SSIM (a, b) = l(\bm {a}, \bm {b}) c(\bm {a}, \bm {b}) s(\bm {a}, \bm {b}) $$
where [S1]:
$$ l(\bm {a}, \bm {b}) = \frac {2\mu _{a}\mu _{b} + C_{1}}{\mu _{a}^{2} + \mu _{b}^{2} + C_{1}}, \quad c(\bm {a}, \bm {b}) = \frac {2\sigma _{a}\sigma _{b} + C_{2}}{\sigma _{a}^{2} + \sigma _{b}^{2} + C_{2}}, \quad s(\bm {a}, \bm {b}) = \frac {\sigma _{ab} + C_{3}}{\sigma _{a}\sigma _{b} + C_{3}} $$
Here, $\mu_a, \mu_b$ are means, $\sigma_a, \sigma_b$ are standard deviations, and $\sigma_{ab}$ is the covariance of the pixel values in the patch vectors $\bm{a}$ and $\bm{b}$ [S1].

**Algorithm Steps:**
1.  Sample a data minibatch of rays $\mathcal{R}$ from the dataset.
2.  Obtain ground-truth pixels $\mathcal{C} = \{\mathbf{C}(\mathbf{r}) | \mathbf{r} \in \mathcal{R}\}$.
3.  Compute rendered pixels $\hat{\mathcal{C}} = \{\hat{\mathbf{C}}(\mathbf{r}) | \mathbf{r} \in \mathcal{R}\}$ using the current network parameters [S1].
4.  Initialize $L_{\SSSIM} = 0$.
5.  Loop $m$ from 1 to $M$:
    a.  Initialize stochastic patch generation function $\mathcal{P}^{(m)}$.
    b.  Transform rendered pixels into rendered stochastic patch $\hat{\mathcal{P}}^{(m)}(\hat{\mathcal{C}})$.
    c.  Transform ground-truth pixels into ground-truth stochastic patch $\mathcal{P}^{(m)}(\mathcal{C})$.
    d.  Compute SSIM between $\hat{\mathcal{P}}^{(m)}(\hat{\mathcal{C}})$ and $\mathcal{P}^{(m)}(\mathcal{C})$ using kernel $K \times K$ and stride $s$ [S1].
    e.  Add the computed SSIM to $L_{\SSSIM}$ [S1].
6.  Average the $M$ computed SSIM values: $\SSSIM = L_{\SSSIM} / M$.
7.  Calculate S3IM loss: $L_{\SSSIM} = 1 - \SSSIM$ [S1].
8.  Calculate conventional loss $L(\Theta)$ (e.g., MSE or L1) [S1].
9.  Calculate total loss $L_{\mathrm{M}}(\Theta) = L(\Theta) + \lambda L_{\SSSIM}$ [S1].
10. Compute gradient $\nabla L_{\mathrm{M}}(\Theta)$ [S1].
11. Update model parameters $\Theta$ using an SGD-like algorithm [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def s3im_loss(rendered_pixels, ground_truth_pixels, batch_size, M, K, lambda_weight):
    """
    Computes the S3IM multiplex loss.
    
    Args:
        rendered_pixels: Array of shape (batch_size, 3) containing rendered RGB values.
        ground_truth_pixels: Array of shape (batch_size, 3) containing ground-truth RGB values.
        batch_size: Number of rays/pixels in the minibatch (B).
        M: Number of stochastic estimations.
        K: Kernel size for SSIM (assumed square KxK).
        lambda_weight: Weight for the S3IM term.
        
    Returns:
        total_loss: The combined MSE and S3IM loss.
    """
    # 1. Compute conventional pixel-wise MSE loss
    mse_loss = 0.0
    for i in range(batch_size):
        diff = rendered_pixels[i] - ground_truth_pixels[i]
        mse_loss += sum(d*d for d in diff)
    mse_loss /= batch_size
    
    # 2. Compute S3IM loss
    ssim_sum = 0.0
    for m in range(M):
        # Step 2a: Initialize stochastic patch generation
        # Randomly select pixels to form a pseudo-image structure.
        # Since pixels are from a minibatch, they lack spatial coherence.
        # We treat the selected subset as a flattened vector or grid for SSIM.
        # The source implies forming a patch P(C) from the batch.
        # To implement SSIM, we need a 2D structure. 
        # The source states: "B rays/pixels ... randomly form a rendered patch".
        # We assume the batch is reshaped or sampled into a grid compatible with KxK.
        # For implementation, we sample indices to form the patch.
        
        # Generate stochastic patch indices (simplified: use all pixels reshaped or sampled)
        # The source uses stride s=K. If B is not a perfect square, padding or truncation is needed.
        # We assume B is sufficient to form at least one KxK patch or the patch is formed by 
        # reshaping the batch into a grid.
        
        # Let's assume we reshape the batch into a grid of size HxW such that we can apply KxK windows.
        # Or, more simply, the "patch" is just the set of pixels. 
        # However, SSIM requires local statistics. The source says: 
        # "compute SSIM with the kernel size KxK and stride size s over the rendered patch".
        # This implies the batch pixels are arranged into a 2D image-like structure.
        
        # Implementation detail: Reshape batch to (H, W, 3) where H*W = B.
        # If B is not a square, we can pad or use a rectangular grid.
        # For simplicity, we assume B is a square number or we take a subset.
        # The source does not specify how to map 1D batch to 2D patch.
        # We will assume a standard reshaping to a square grid if possible, 
        # or a rectangular grid H x W.
        
        # Let's assume we reshape to a grid.
        side = int(round(batch_size ** 0.5))
        # Pad if necessary to make it square for simple reshaping, or use H, W
        # The source doesn't specify padding. We'll assume the batch fits or is reshaped.
        # To be robust, we can just take the first K*K pixels if B >= K*K, 
        # but the source says "B pixels ... form a patch".
        # We will reshape the entire batch into a 2D grid.
        
        # Reshape to 2D
        # Note: This is an implementation choice not explicitly detailed in the source,
        # but necessary to apply 2D SSIM.
        H = int(batch_size ** 0.5)
        W = batch_size // H
        # If not divisible, we might need to handle remainder. 
        # For the purpose of the article, we assume a valid grid formation.
        
        rendered_patch = rendered_pixels.reshape(H, W, 3)
        gt_patch = ground_truth_pixels.reshape(H, W, 3)
        
        # Compute SSIM on this patch
        # SSIM is computed locally with KxK kernel and stride s=K
        ssim_val = compute_local_ssim(rendered_patch, gt_patch, K, K)
        ssim_sum += ssim_val
        
    avg_ssim = ssim_sum / M
    s3im_loss = 1.0 - avg_ssim
    
    # 3. Total Loss
    total_loss = mse_loss + lambda_weight * s3im_loss
    return total_loss

def compute_local_ssim(img1, img2, K, s):
    """
    Computes the mean SSIM over a 2D image using a sliding window.
    """
    H, W, C = img1.shape
    ssim_map = []
    
    # Constants
    C1 = (0.01 * 1.0)**2
    C2 = (0.03 * 1.0)**2
    C3 = C2 / 2.0
    
    # Iterate over the image with stride s
    for i in range(0, H - K + 1, s):
        for j in range(0, W - K + 1, s):
            # Extract KxK windows
            win1 = img1[i:i+K, j:j+K, :]
            win2 = img2[i:i+K, j:j+K, :]
            
            # Compute local statistics
            mu1 = mean(win1)
            mu2 = mean(win2)
            
            sigma1_sq = variance(win1)
            sigma2_sq = variance(win2)
            sigma12 = covariance(win1, win2)
            
            # Compute SSIM components
            l = (2 * mu1 * mu2 + C1) / (mu1**2 + mu2**2 + C1)
            c = (2 * sigma1_sq * sigma2_sq + C2) / (sigma1_sq + sigma2_sq + C2)
            s_val = (sigma12 + C3) / (sqrt(sigma1_sq) * sqrt(sigma2_sq) + C3)
            
            ssim_map.append(l * c * s_val)
            
    return mean(ssim_map)

def mean(tensor):
    return sum(tensor.flatten()) / len(tensor.flatten())

def variance(tensor):
    m = mean(tensor)
    return sum((x - m)**2 for x in tensor.flatten()) / len(tensor.flatten())

def covariance(tensor1, tensor2):
    m1 = mean(tensor1)
    m2 = mean(tensor2)
    return sum((x - m1) * (y - m2) for x, y in zip(tensor1.flatten(), tensor2.flatten())) / len(tensor1.flatten())
```

**Edge Cases and Pitfalls:**
*   **Stochastic Sampling:** Stochastic sampling breaks positional relationships. The method accounts for this by treating sampled pixels as a 'stochastic patch' rather than a spatially coherent image patch [S1].
*   **Stride:** Non-overlapping strides ($s=K$) are used to ensure independent samples within the batch [S1].
*   **Direct SSIM Optimization:** Directly optimizing standard SSIM via local patches may not work well and increases coding/computational costs. S3IM outperforms this by capturing non-local structural information from distant pixels across different images [S1].
*   **Batch Size:** The batch size $B$ must be sufficient to form the stochastic patches. Typical value is 1024 [S1].

## Operating characteristics
*   **Time Complexity:** The method adds $M$ SSIM computations per minibatch. Since SSIM is computed over local windows, the cost is proportional to the number of pixels in the batch and the number of stochastic estimations $M$. The source states the extra computational cost is limited [S1].
*   **Memory Complexity:** Requires storing the minibatch of rays and rendered pixels. No additional large data structures are needed beyond the batch [S1].
*   **Measured Performance:**
    *   On Replica Dataset with DVGO: Mean PSNR improved from 17.07 to 33.50, SSIM from 0.696 to 0.955, LPIPS from 0.510 to 0.0637 [S1].
    *   On Replica Dataset with TensoRF: Mean PSNR improved from 14.30 to 39.05, SSIM from 0.574 to 0.971, LPIPS from 0.689 to 0.0454 [S1].
    *   On T&T Dataset with DVGO: Mean PSNR improved from 22.42 to 23.20, SSIM from 0.776 to 0.809, LPIPS from 0.236 to 0.176 [S1].
    *   With 20% training images (Sparse Truck): PSNR improved by up to 4.32 [S1].
    *   On Dynamic Scene 'Lego' with D-NeRF: PSNR improved from 21.29 to 23.36, SSIM from 0.821 to 0.900, LPIPS from 0.0634 to 0.0482 [S1].
*   **Strengths:** Improves reconstruction quality for complex details and sparse inputs. Robust to image corruption and noise. Improves geometric quality metrics (e.g., Chamfer Distance) in addition to RGB metrics [S1].
*   **Limitations:** Requires fine-tuning of the hyperparameter $\lambda$ [S1].
*   **When to Use:** When standard point-wise losses (MSE) fail to capture structural details, especially with sparse inputs or noisy data. Applicable to NeRF, DVGO, TensoRF, D-NeRF, and NeuS [S1].

## Background
S3IM was introduced in the paper "S3IM: Stochastic Structural SIMilarity and Its Unreasonable Effectiveness for Neural Fields" [S1]. It is a model-agnostic training paradigm orthogonal to existing point-wise training paradigms and NeRF variants [S1]. It was designed to address the limitation of conventional losses that do not utilize collective supervision of distant pixels [S1].

## Sources
[S1] S3IM: Stochastic Structural SIMilarity and Its Unreasonable Effectiveness for Neural Fields, https://doi.org/10.1109/iccv51070.2023.01652, fetched 2026-10-26
[S2] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
