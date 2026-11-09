# S3IM (Stochastic Structural SIMilarity)

Also called: Multiplex Loss, S3IM loss

## Overview
S3IM is a regularization term for neural field optimization that captures nonlocal structural similarity by processing multiple data points (pixels/rays) as a collective set rather than individually [S1]. It addresses the limitation of point-wise losses like Mean Squared Error (MSE), which fail to utilize the collective supervision of distant pixels that provide rich structural information [S1]. Unlike conventional pixel-wise losses, S3IM cannot be expressed as the sum of any loss computed over individual pixels from the batch, allowing it to enforce global structural consistency at nearly no additional computational cost [S1].

## How it works
S3IM integrates into the total loss function as a multiplex term, combining standard photometric reconstruction with a stochastic estimation of Structural Similarity (SSIM) [S1]. The method operates on a minibatch of rays, treating the sampled pixels as independent "stochastic patches" to bypass the lack of spatial coherence in randomly sampled ray batches [S1].

### Inputs and Outputs
*   **Inputs**:
    *   $\Theta$: Model parameters (e.g., NeRF MLP weights).
    *   $\mathcal{R}$: A minibatch of $B$ rays sampled from the dataset.
    *   $\mathcal{C}$: Ground-truth pixel colors corresponding to $\mathcal{R}$, where $\mathcal{C} = \{\mathbf{C}(\mathbf{r}) | \mathbf{r} \in \mathcal{R}\}$.
    *   $\hat{\mathcal{C}}$: Rendered pixel colors corresponding to $\mathcal{R}$, where $\hat{\mathcal{C}} = \{\hat{\mathbf{C}}(\mathbf{r}) | \mathbf{r} \in \mathcal{R}\}$.
    *   $M$: Number of stochastic estimations (iterations).
    *   $K$: Kernel size for SSIM computation (typically $K \times K$).
    *   $\lambda$: Hyperparameter weighting the S3IM loss relative to MSE.
*   **Outputs**:
    *   $\mathcal{L}_{\text{S3IM}}$: The scalar S3IM loss value.
    *   $\mathcal{L}_{\text{total}}$: The combined loss for backpropagation.

### Data Structures
*   **Minibatch $\mathcal{R}$**: A list or tensor of $B$ rays. Each ray $\mathbf{r}$ contains origin $\mathbf{o}$, direction $\mathbf{d}$, and associated ground-truth color $\mathbf{C}$.
*   **Pixel Tensors**: $\mathcal{C}$ and $\hat{\mathcal{C}}$ are tensors of shape $(B, 3)$ (for RGB) or $(B, C)$ for $C$ channels.
*   **Stochastic Patch $\mathcal{P}^{(m)}$**: A conceptual grouping of pixels. Since the input is a random batch, the "patch" is formed by reshaping or viewing the batch elements as a grid of size $K \times K$ (assuming $B$ is a multiple of $K^2$) or by treating the entire batch as a single large stochastic patch if $B < K^2$ is not the case. The source specifies forming a patch $\mathcal{P}$ from $B$ pixels [S1].

### Parameters
*   **$B$ (Batch Size)**: Number of rays/pixels per minibatch. Typical value: 1024 [S1].
*   **$K$ (Kernel Size)**: Size of the local window for SSIM computation. Typical value: $4 \times 4$ [S1].
*   **$M$ (Iterations)**: Number of times the stochastic patch generation and SSIM computation are repeated. Default value: 10 [S1].
*   **$\lambda$**: Weight for S3IM. Not given in the sources.
*   **SSIM Constants**:
    *   $K_1 = 0.01$
    *   $K_2 = 0.03$
    *   $L = 1$ (for RGB images)
    *   $C_1 = (K_1 L)^2 = 0.0001$
    *   $C_2 = (K_2 L)^2 = 0.0009$
    *   $C_3 = C_2 / 2 = 0.00045$ [S1]

### Formulas
The total multiplex loss is defined as [S1]:
$$ \mathcal{L}_{\mathrm{M}} (\Theta ) = \mathcal{L}_{\mathrm{MSE}} (\Theta ) + \lambda \mathcal{L}_{\mathrm{S3IM}} (\Theta ) $$

Where $\mathcal{L}_{\mathrm{MSE}}$ is the standard pixel-wise MSE:
$$ \mathcal{L}_{\mathrm{MSE}} (\Theta ) = \frac {1}{\|\mathcal {R}\|}\sum _{\bm {r}\in \mathcal {R}} || \hat{\mathbf{C}}(\bm {r}) - \mathbf{C}(\bm {r}) ||_2^2 $$

The S3IM loss is derived from the S3IM index, which is the average of $M$ SSIM estimators over stochastic patches [S1]:
$$ \mathcal{L}_{\mathrm{S3IM}} (\Theta ) = 1 - \SSSIM (\hat {\mathcal {C}},\mathcal {C}) $$
$$ \SSSIM (\hat {\mathcal {C}},\mathcal {C}) = \frac {1}{M} \sum _{m=1}^{M} \SSIM (\mathcal{P}^{(m)}(\hat {\mathcal {C}}), \mathcal{P}^{(m)}(\mathcal {C})) $$

The SSIM index between two patches $\mathbf{a}$ and $\mathbf{b}$ is the product of luminance, contrast, and structure terms [S1]:
$$ \SSIM (\mathbf{a}, \mathbf{b}) = l(\mathbf{a}, \mathbf{b}) \cdot c(\mathbf{a}, \mathbf{b}) \cdot s(\mathbf{a}, \mathbf{b}) $$

The comparison terms are [S1]:
$$ l(\mathbf{a}, \mathbf{b}) = \frac {2\mu _{a}\mu _{b} + C_{1}}{\mu _{a}^{2} + \mu _{b}^{2} + C_{1}} $$
$$ c(\mathbf{a}, \mathbf{b}) = \frac {2\sigma _{a}\sigma _{b} + C_{2}}{\sigma _{a}^{2} + \sigma _{b}^{2} + C_{2}} $$
$$ s(\mathbf{a}, \mathbf{b}) = \frac {\sigma _{ab} + C_{3}}{\sigma _{a}\sigma _{b} + C_{3}} $$

Where:
*   $\mu_a, \mu_b$: Mean intensities of patches $\mathbf{a}, \mathbf{b}$.
*   $\sigma_a, \sigma_b$: Standard deviations of patches $\mathbf{a}, \mathbf{b}$.
*   $\sigma_{ab}$: Covariance between patches $\mathbf{a}, \mathbf{b}$.

### Numbered Steps
1.  **Sample Minibatch**: Sample a minibatch of rays $\mathcal{R}$ from the dataset. Obtain ground-truth pixels $\mathcal{C}$ and compute rendered pixels $\hat{\mathcal{C}}$ [S1].
2.  **Stochastic Estimation Loop**: Initialize a sum variable for SSIM. Loop $m$ from 1 to $M$:
    a.  **Generate Stochastic Patch**: Transform the rendered pixels $\hat{\mathcal{C}}$ and ground-truth pixels $\mathcal{C}$ into stochastic patches $\mathcal{P}^{(m)}(\hat{\mathcal{C}})$ and $\mathcal{P}^{(m)}(\mathcal{C})$. This involves arranging the $B$ pixels into a structure compatible with the $K \times K$ kernel. The source states that pixels are randomly formed into a patch [S1].
    b.  **Compute SSIM**: Compute the SSIM index between $\mathcal{P}^{(m)}(\hat{\mathcal{C}})$ and $\mathcal{P}^{(m)}(\mathcal{C})$ using kernel size $K \times K$ and stride $s=K$. Note: Since the pixels are from a random batch, they do not form a spatially coherent image. The method treats them as a "stochastic patch" where the local statistics are computed over the subset of pixels defined by the kernel window within this stochastic arrangement [S1].
    c.  **Accumulate**: Add the computed SSIM value to the sum.
3.  **Average**: Divide the sum by $M$ to obtain $\SSSIM (\hat {\mathcal {C}},\mathcal {C})$ [S1].
4.  **Calculate Loss**: Compute $\mathcal{L}_{\mathrm{S3IM}} = 1 - \SSSIM (\hat {\mathcal {C}},\mathcal {C})$ [S1].
5.  **Combine**: Compute total loss $\mathcal{L}_{\mathrm{M}} = \mathcal{L}_{\mathrm{MSE}} + \lambda \mathcal{L}_{\mathrm{S3IM}}$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_s3im_loss(rendered_colors, ground_truth_colors, batch_size, kernel_size, num_iterations, lambda_weight):
    """
    Computes the S3IM loss and total multiplex loss.
    
    Args:
        rendered_colors: Tensor of shape (batch_size, channels), rendered pixel colors.
        ground_truth_colors: Tensor of shape (batch_size, channels), ground truth pixel colors.
        batch_size: Number of rays in the minibatch (B).
        kernel_size: Size of the SSIM kernel (K).
        num_iterations: Number of stochastic estimations (M).
        lambda_weight: Weight for the S3IM term (lambda).
        
    Returns:
        total_loss: Scalar tensor, the combined MSE and S3IM loss.
    """
    
    # Constants for SSIM
    K1 = 0.01
    K2 = 0.03
    L = 1.0
    C1 = (K1 * L) ** 2
    C2 = (K2 * L) ** 2
    C3 = C2 / 2.0
    
    # 1. Compute MSE Loss
    # MSE is computed pixel-wise over the batch
    mse_loss = torch.mean((rendered_colors - ground_truth_colors) ** 2)
    
    # 2. Compute S3IM Loss
    s3im_sum = 0.0
    
    for m in range(num_iterations):
        # Step 2a: Generate Stochastic Patch
        # The source describes forming a patch from the batch pixels.
        # Since the batch is random, we treat the batch as a set of pixels.
        # To apply a KxK kernel, we typically need to reshape or select subsets.
        # The source implies using the entire batch as a "stochastic patch" 
        # or reshaping it to fit the kernel logic. 
        # Given B=1024 and K=4, we can reshape to (K*K, B/(K*K)) or similar.
        # However, the source says "stride size s=K" and "kernel size KxK".
        # In standard SSIM, this slides over an image. Here, the "image" is the stochastic arrangement.
        # A common implementation for S3IM (as per the paper's Algorithm 1) 
        # is to simply compute SSIM on the batch viewed as a grid if possible, 
        # or more accurately, the paper states: "randomly form a rendered patch... 
        # compute SSIM with kernel size KxK and stride size s".
        # If B is not a perfect square of K, this is ambiguous. 
        # However, the key insight is that the "patch" is the set of pixels itself.
        # For implementation, we assume the batch is reshaped into a 2D grid 
        # of size (sqrt(B), sqrt(B)) or similar, OR that the SSIM is computed 
        # over the whole batch as a single "patch" if B < K^2, or multiple patches.
        # The most robust interpretation of "stochastic patch" with stride K 
        # on a batch of size B is to treat the batch as a 1D sequence or 
        # reshape to a 2D image of size (B, 1) or (1, B) is not standard.
        # Let's follow the source's implication: The pixels are arranged into a patch.
        # If B=1024, we can reshape to (32, 32). Then apply KxK kernel.
        # But the source says "stride size s=K".
        # Let's assume the standard SSIM calculation on the reshaped batch.
        
        # Reshape batch to 2D image-like structure for SSIM calculation
        # Assuming channels are handled by SSIM implementation or flattened.
        # Standard SSIM takes H, W, C.
        # Let's assume we reshape to (H, W, C) where H*W = B.
        # For simplicity and consistency with "stochastic patch", 
        # we reshape to a square if possible, or use the whole batch as one patch 
        # if the SSIM function supports global stats.
        # The source mentions "local statistics... computed within a local KxK kernel window".
        # This implies sliding window.
        
        # Implementation detail: Reshape to (H, W, C)
        # H = int(sqrt(B))
        # W = int(sqrt(B))
        # If B is not a perfect square, this is a pitfall. 
        # The source does not specify handling non-square B. 
        # We assume B is chosen such that it fits, or we pad.
        # [G] Guard: Ensure batch_size is a perfect square for simple reshaping, 
        # or use a 1D sliding window if the data is treated as a sequence.
        # Given the "stochastic" nature, treating it as a 2D grid is the standard interpretation.
        
        H = int(batch_size ** 0.5)
        W = int(batch_size ** 0.5)
        
        # Check if B is a perfect square
        if H * W != batch_size:
            # [G] Fallback: If not a perfect square, pad or use 1D. 
            # For this article, we assume B is a perfect square as per typical S3IM setups.
            raise ValueError("Batch size must be a perfect square for standard S3IM reshaping.")
            
        # Reshape to (H, W, C)
        # Note: rendered_colors and ground_truth_colors are (B, C)
        # We need to ensure the channel dimension is preserved.
        # If C=3, shape is (B, 3). Reshape to (H, W, 3).
        rendered_patch = rendered_colors.reshape(H, W, -1)
        ground_truth_patch = ground_truth_colors.reshape(H, W, -1)
        
        # Compute SSIM on this patch
        # The SSIM function computes local statistics with kernel K and stride K.
        # Since the pixels are random, the "local" neighborhood is arbitrary.
        ssim_value = compute_ssim(rendered_patch, ground_truth_patch, kernel_size=kernel_size, stride=kernel_size, C1=C1, C2=C2, C3=C3)
        
        s3im_sum += ssim_value
        
    # Average over M iterations
    s3im_index = s3im_sum / num_iterations
    
    # S3IM Loss is 1 - SSIM
    s3im_loss = 1.0 - s3im_index
    
    # Total Loss
    total_loss = mse_loss + lambda_weight * s3im_loss
    
    return total_loss

def compute_ssim(img1, img2, kernel_size, stride, C1, C2, C3):
    """
    Computes SSIM using a sliding window.
    """
    # Get image dimensions
    H, W, C = img1.shape
    
    # Initialize sum of SSIM values and count
    ssim_sum = 0.0
    count = 0
    
    # Sliding window
    for i in range(0, H - kernel_size + 1, stride):
        for j in range(0, W - kernel_size + 1, stride):
            # Extract patches
            patch1 = img1[i:i+kernel_size, j:j+kernel_size, :]
            patch2 = img2[i:i+kernel_size, j:j+kernel_size, :]
            
            # Compute local statistics
            # Flatten patches to compute stats over all channels or per channel?
            # Standard SSIM is often computed per channel and averaged, or on grayscale.
            # The source defines SSIM for vectors a, b.
            # We compute mean, var, cov for the patch.
            
            # Flatten patch to (N, C) where N = K*K
            p1_flat = patch1.reshape(-1, C)
            p2_flat = patch2.reshape(-1, C)
            
            # Compute means
            mu1 = torch.mean(p1_flat, dim=0)
            mu2 = torch.mean(p2_flat, dim=0)
            
            # Compute variances and covariance
            # Covariance matrix is C x C. SSIM structure term uses scalar cov?
            # The source formula s(a,b) uses sigma_ab.
            # If C > 1, this is ambiguous. Usually, SSIM is computed per channel 
            # and averaged, or the images are converted to grayscale.
            # [G] Assumption: Compute SSIM per channel and average, or assume C=1 for simplicity.
            # Given RGB, we compute per channel.
            
            channel_ssims = []
            for c in range(C):
                a = p1_flat[:, c]
                b = p2_flat[:, c]
                
                mu_a = torch.mean(a)
                mu_b = torch.mean(b)
                
                sigma_a_sq = torch.var(a, unbiased=False)
                sigma_b_sq = torch.var(b, unbiased=False)
                sigma_ab = torch.mean((a - mu_a) * (b - mu_b))
                
                # Luminance
                l = (2 * mu_a * mu_b + C1) / (mu_a**2 + mu_b**2 + C1)
                
                # Contrast
                sigma_a = torch.sqrt(sigma_a_sq)
                sigma_b = torch.sqrt(sigma_b_sq)
                c_term = (2 * sigma_a * sigma_b + C2) / (sigma_a_sq + sigma_b_sq + C2)
                
                # Structure
                s = (sigma_ab + C3) / (sigma_a * sigma_b + C3)
                
                channel_ssims.append(l * c_term * s)
                
            ssim_sum += torch.mean(torch.stack(channel_ssims))
            count += 1
            
    return ssim_sum / count
```

### Edge Cases and Pitfalls
*   **Stochastic Sampling Breaks Positional Relationships**: The method accounts for this by treating sampled pixels as a "stochastic patch" rather than a spatially coherent image patch [S1].
*   **Stride and Independence**: The stride size $s$ is set equal to the kernel size $K$ to ensure independent samples within the batch, avoiding overlapping pixels in the stochastic patch calculation [S1].
*   **Batch Size Constraints**: The pseudocode assumes the batch size $B$ is a perfect square to facilitate reshaping into a 2D grid for SSIM computation. If $B$ is not a perfect square, padding or a different arrangement is required [G].
*   **Channel Handling**: The SSIM formula is defined for vectors. For RGB images, SSIM is typically computed per channel and averaged, or the image is converted to grayscale. The source does not explicitly specify this, but it is a standard implementation detail [G].

## Operating characteristics
*   **Time Complexity**: The S3IM loss requires $M$ iterations of SSIM computation. SSIM itself is $O(B \cdot K^2)$ per iteration if computed naively, or $O(B)$ with optimized sliding window. Thus, S3IM adds a factor of $M$ to the computational cost of the loss calculation. With $M=10$ and $K=4$, the overhead is small but non-negligible compared to MSE [S1].
*   **Memory Complexity**: Requires storing the batch $\mathcal{R}$, rendered colors $\hat{\mathcal{C}}$, and ground truth $\mathcal{C}$. No additional large data structures are needed beyond the batch itself [S1].
*   **Performance**: The source reports that S3IM improves quality metrics significantly, e.g., test MSE loss drops by more than 90% for TensoRF and DVGO over eight novel view synthesis tasks, and provides a 198% F-score gain and 64% Chamfer L1 distance reduction for surface representation [S1].
*   **Strengths**: Captures nonlocal structural similarity; improves NeRF and neural surface representation quality; nearly free computational cost compared to the gain in quality [S1].
*   **Limitations**: Relies on stochastic sampling, which may introduce variance (mitigated by $M$ iterations); requires careful handling of batch size and reshaping [S1].

## Background
S3IM was introduced in the paper "S3IM: Stochastic Structural SIMilarity and Its Unreasonable Effectiveness for Neural Fields" [S1]. The authors state that previous research failed to use the collective supervision of distant pixels, which provide rich structural information [S1]. S3IM is the first nonlocal multiplex training paradigm for NeRF and relevant neural field methods, processing multiple data points as a whole set instead of independently [S1].

## Sources
[S1] S3IM: Stochastic Structural SIMilarity and Its Unreasonable Effectiveness for Neural Fields, https://doi.org/10.1109/iccv51070.2023.01652, fetched 2026-10-26
