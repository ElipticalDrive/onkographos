# S3IM (Stochastic Structural SIMilarity)

Also called: Multiplex Loss, S3IM loss

## Overview
S3IM is a multiplex regularization term for neural fields that captures nonlocal structural similarity by processing multiple data points (pixels/rays) as a whole set, rather than individually [S1]. It addresses the limitation of point-wise losses (like MSE) which fail to utilize the collective supervision of distant pixels [S1]. Unlike conventional MSE loss, S3IM cannot be expressed as the sum of any other loss computed over individual pixels from the batch [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A minibatch of rays $\mathcal{R}$, ground-truth pixel colors $\mathcal{C} = \{\mathbf{C}(\mathbf{r}) | \mathbf{r} \in \mathcal{R}\}$, rendered pixel colors $\hat{\mathcal{C}} = \{\hat{\mathbf{C}}(\mathbf{r}) | \mathbf{r} \in \mathcal{R}\}$, and model parameters $\Theta$.
*   **Outputs:** The S3IM loss value $L_{\text{S3IM}}(\Theta)$ and the total multiplex loss $L_{\text{M}}(\Theta)$.

**Data Structures:**
*   $\mathcal{R}$: Set of rays in the minibatch. Size $B$ (batch size).
*   $\mathcal{C}, \hat{\mathcal{C}}$: Vectors of color values corresponding to rays in $\mathcal{R}$.
*   $\mathcal{P}^{(m)}$: Stochastic patch generation function for iteration $m$.

**Parameters:**
*   $B$: Batch size (number of rays/pixels per minibatch). Typical value: 1024 [S1].
*   $K$: Kernel size for SSIM computation. Typical value: $4 \times 4$ [S1].
*   $s$: Stride size for SSIM sliding window. Typical value: $K$ (non-overlapping) [S1].
*   $M$: Number of stochastic estimations (iterations). Typical value: 10 [S1].
*   $\lambda$: Hyperparameter adjusting the importance of S3IM relative to MSE [S1].
*   $K_1, K_2$: SSIM constants. Typical values: $K_1 = 0.01, K_2 = 0.03$ [S1].
*   $L$: Dynamic range of pixel values. Typical value: 1 for normalized RGB [S1].

**Formulas:**
The total multiplex loss is defined as [S1]:
$$ L_{\mathrm {M}} (\Theta ) = \frac {1}{\|\mathcal {R}\|}\sum _{\bm {r}\in \mathcal {R}} l_{\MSE }(\Theta , \bm {r}) + \lambda L_{\SSSIM }(\Theta , \mathcal {R}) $$
where $l_{\MSE }$ is the per-pixel MSE and $L_{\SSSIM }$ is the S3IM loss.

The S3IM value is computed as the average of $M$ SSIM estimators over stochastic patches [S1]:
$$ L_{\SSSIM }(\Theta , \mathcal {R}) = 1 - \SSSIM (\hat {\mathcal {C}},\mathcal {C}) = 1 - \frac {1}{M} \sum _{m=1}^{M} \SSIM (\mathcal {P}^{(m)}(\hat {\mathcal {C}}), \mathcal {P}^{(m)}(\mathcal {C})) $$
Note: The source defines $\SSSIM$ as the similarity score, but the loss $L_{\text{S3IM}}$ is typically $1 - \text{Similarity}$ to minimize dissimilarity. Algorithm 1 in [S1] explicitly computes $L_{\text{S3IM}} = 1 - \frac{1}{M}\sum \SSIM$.

SSIM is defined as the product of luminance, contrast, and structure terms [S1]:
$$ \SSIM (a, b) = l(\bm {a}, \bm {b}) c(\bm {a}, \bm {b}) s(\bm {a}, \bm {b}) $$
where $a$ and $b$ are the flattened vectors of pixel values in the stochastic patches [S1].

The luminance, contrast, and structure comparison terms are [S1]:
$$ l(\bm {a}, \bm {b}) = \frac {2\mu _{a}\mu _{b} + C_{1}}{\mu _{a}^{2} + \mu _{b}^{2} + C_{1}} $$
$$ c(\bm {a}, \bm {b}) = \frac {2\sigma _{a}\sigma _{b} + C_{2}}{\sigma _{a}^{2} + \sigma _{b}^{2} + C_{2}} $$
$$ s(\bm {a}, \bm {b}) = \frac {\sigma _{ab} + C_{3}}{\sigma _{a}\sigma _{b} + C_{3}} $$

Constants for SSIM [S1]:
$$ C_1 = (K_1 L)^2, \quad C_2 = (K_2 L)^2, \quad C_3 = C_2 / 2 $$

**Steps:**
1.  Sample a data minibatch of rays $\mathcal{R}$ from the dataset. Obtain ground-truth pixels $\mathcal{C}$ and compute rendered pixels $\hat{\mathcal{C}}$ [S1].
2.  Loop $m$ from 1 to $M$:
    a. Initialize stochastic patch generation function $\mathcal{P}^{(m)}$. This function randomly selects pixels from $\mathcal{C}$ and $\hat{\mathcal{C}}$ to form patches [S1].
    b. Transform rendered pixels into rendered stochastic patch $\mathcal{P}^{(m)}(\hat{\mathcal{C}})$ and ground-truth pixels into ground-truth stochastic patch $\mathcal{P}^{(m)}(\mathcal{C})$ [S1].
    c. Compute SSIM between $\mathcal{P}^{(m)}(\hat{\mathcal{C}})$ and $\mathcal{P}^{(m)}(\mathcal{C})$ using kernel size $K \times K$ and stride $s=K$ [S1].
3.  Average the $M$ computed SSIM values to obtain the final S3IM similarity score [S1].
4.  Compute S3IM loss as $1 - \text{Average SSIM}$ [S1].
5.  Compute total multiplex loss $L_{\text{M}}$ using the formula above [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_s3im_loss(rendered_colors, gt_colors, batch_size, kernel_size, num_iterations, lambda_weight):
    """
    Computes the S3IM multiplex loss.
    
    Args:
        rendered_colors: Array of shape (batch_size, channels) containing rendered pixel colors.
        gt_colors: Array of shape (batch_size, channels) containing ground truth pixel colors.
        batch_size: Number of rays/pixels in the minibatch (B).
        kernel_size: Size of the local window for SSIM (K).
        num_iterations: Number of stochastic estimations (M).
        lambda_weight: Hyperparameter for S3IM importance (lambda).
        
    Returns:
        total_loss: The total multiplex loss value.
    """
    # Step 1: Compute standard MSE loss
    mse_per_pixel = mean_squared_error(rendered_colors, gt_colors)
    mse_loss = mean(mse_per_pixel)
    
    # Step 2: Compute S3IM loss
    ssim_sum = 0.0
    
    for m in range(num_iterations):
        # Step 2a & 2b: Stochastic patch generation
        # Randomly sample indices to form patches. 
        # Since pixels are stochastic, we treat the batch as a pool.
        # We need to form patches of size K*K. 
        # The source implies we form patches from the batch pixels directly.
        # If batch_size < K*K, this step is undefined or requires padding/repetition.
        # Assuming batch_size >= K*K.
        
        # Flatten colors for patch extraction
        flat_rendered = rendered_colors.flatten()
        flat_gt = gt_colors.flatten()
        
        # Generate random indices for the patch
        # The source says "randomly form a rendered patch... and corresponding ground-truth patch"
        # It implies selecting a subset of pixels from the batch to form the patch vectors a and b.
        # However, SSIM requires spatial correspondence. 
        # The source states: "stochastically sampled pixels... lose the positional relationship completely."
        # And "using non-overlapping strides (s=K) to ensure independent samples".
        # This implies we are not extracting spatially contiguous patches from an image, 
        # but rather forming two vectors of equal length from the batch pixels.
        # To compute SSIM, we need paired vectors. 
        # The most logical interpretation of "stochastic patch" in this context, 
        # given the loss of positional info, is to randomly pair pixels or 
        # simply take the first K*K pixels as a "patch" vector if the batch is treated as a flat list.
        # However, Algorithm 1 says "Transform... into stochastic patch".
        # Let's assume we select a subset of indices to form the patch vectors.
        # To ensure independence and non-overlap across the M iterations or within the batch,
        # we can simply take the first K*K elements as the patch for this iteration, 
        # or randomly sample K*K indices. 
        # Given "stride s=K" and "non-overlapping", it suggests a sliding window approach 
        # on the *flattened* batch list, but since it's stochastic, we shuffle first.
        
        # Shuffle indices to randomize the "spatial" arrangement in the flat list
        indices = random_permutation(batch_size * channels)
        
        # Extract patch vectors (flattened)
        # We need vectors of length K*K. 
        # If the batch is large, we might only use a subset. 
        # The source says "B pixels per minibatch... choose kernel size KxK".
        # It does not explicitly say we use the whole batch for one SSIM calculation.
        # It says "compute SSIM... over the rendered patch".
        # Let's assume we form one patch of size K*K from the batch for each iteration.
        
        patch_indices = indices[:kernel_size * kernel_size]
        
        patch_rendered = flat_rendered[patch_indices]
        patch_gt = flat_gt[patch_indices]
        
        # Step 2c: Compute SSIM for this patch
        ssim_val = compute_ssim(patch_rendered, patch_gt, kernel_size)
        ssim_sum += ssim_val
        
    # Step 3: Average SSIM
    avg_ssim = ssim_sum / num_iterations
    
    # Step 4: S3IM Loss
    s3im_loss = 1.0 - avg_ssim
    
    # Step 5: Total Loss
    total_loss = mse_loss + lambda_weight * s3im_loss
    
    return total_loss

def compute_ssim(a, b, kernel_size):
    """
    Computes SSIM between two flattened vectors a and b.
    Note: Standard SSIM is computed over a 2D grid. 
    The source applies SSIM to "stochastic patches". 
    If the patch is just a vector, we treat it as a 1D signal or reshape to sqrt(K*K)x sqrt(K*K).
    Given K=4, we reshape to 4x4.
    """
    # Reshape to 2D
    side = int(sqrt(kernel_size * kernel_size))
    a_2d = reshape(a, (side, side))
    b_2d = reshape(b, (side, side))
    
    # Compute local statistics within the KxK window (which is the whole patch here)
    # Since the patch IS the window, we compute global stats for this patch.
    mu_a = mean(a_2d)
    mu_b = mean(b_2d)
    sigma_a = std(a_2d)
    sigma_b = std(b_2d)
    sigma_ab = covariance(a_2d, b_2d)
    
    # Constants
    K1 = 0.01
    K2 = 0.03
    L = 1.0
    C1 = (K1 * L)**2
    C2 = (K2 * L)**2
    C3 = C2 / 2.0
    
    # Luminance
    l = (2 * mu_a * mu_b + C1) / (mu_a**2 + mu_b**2 + C1)
    
    # Contrast
    c = (2 * sigma_a * sigma_b + C2) / (sigma_a**2 + sigma_b**2 + C2)
    
    # Structure
    s = (sigma_ab + C3) / (sigma_a * sigma_b + C3)
    
    return l * c * s
```

**Edge Cases and Pitfalls:**
*   **Stochastic Sampling:** Stochastic sampling breaks positional relationships. The method accounts for this by treating sampled pixels as a 'stochastic patch' rather than a spatially coherent image patch [S1].
*   **Stride:** Non-overlapping strides ($s=K$) are used to ensure independent samples within the batch [S1].
*   **Batch Size:** If the batch size $B$ is smaller than the kernel size $K \times K$, the patch generation is undefined. Typical batch size 1024 and kernel 4x16 ensures $B \gg K^2$ [S1].

## Operating characteristics
*   **Time Complexity:** The S3IM loss requires $M$ iterations of SSIM computation per batch. SSIM is $O(K^2)$ per patch. Total complexity per batch is $O(M \cdot K^2)$. This is added to the standard MSE computation.
*   **Memory Complexity:** Requires storing the batch of rays and rendered pixels. No additional large data structures are needed beyond the patch vectors.
*   **Performance:** S3IM improves quality metrics significantly, e.g., test MSE loss drops by more than 90% for TensoRF and DVGO over eight novel view synthesis tasks [S1].
*   **Strengths:** Captures nonlocal structural similarity; improves NeRF and neural surface representation quality; "unreasonable effectiveness" for difficult tasks [S1].
*   **Limitations:** Adds computational overhead due to $M$ iterations; relies on stochastic sampling which may introduce variance (mitigated by averaging over $M$) [S1].
*   **When to use:** When structural similarity is more important than pixel-wise accuracy; for difficult tasks like novel view synthesis where global structure matters [S1].
*   **When not to use:** When computational resources are extremely limited and $M$ iterations are prohibitive; when pixel-wise fidelity is the sole objective and structural coherence is irrelevant.

## Background
S3IM was introduced in the paper "S3IM: Stochastic Structural SIMilarity and Its Unreasonable Effectiveness for Neural Fields" [S1]. It is the first multiplex training paradigm for NeRF and relevant neural field methods that processes multiple data points as a whole set instead of independently [S1]. The method was developed to address the failure of point-wise losses to use collective supervision of distant pixels [S1].

## Sources
[S1] S3IM: Stochastic Structural SIMilarity and Its Unreasonable Effectiveness for Neural Fields, https://doi.org/10.1109/iccv51070.2023.01652, fetched 2026-10-26
