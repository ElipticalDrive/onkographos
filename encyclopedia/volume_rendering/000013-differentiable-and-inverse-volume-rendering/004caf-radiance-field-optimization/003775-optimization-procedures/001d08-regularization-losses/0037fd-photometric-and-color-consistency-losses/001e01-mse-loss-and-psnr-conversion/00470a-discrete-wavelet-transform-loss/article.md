# Discrete Wavelet Transform Loss

Also called: DW loss, Discrete Wavelet Loss, LDW

## Overview
The Discrete Wavelet Transform (DWT) Loss is a regularization term for Neural Radiance Fields (NeRF) that minimizes the difference between the frequency representations of rendered and ground-truth images by decomposing them into sub-bands [S1, unofficial]. It addresses high-frequency overfitting in few-shot NeRF training by explicitly prioritizing low-frequency learning during early stages [S1, unofficial]. Unlike its sibling [MSE Loss and PSNR Conversion](1651-mse-loss-and-psnr-conversion.md), which compares pixel-wise values globally, the DW loss operates on local structures using patch-based sampling to supervise low and mid-frequency components [S1, unofficial].

## How it works
The method decomposes rendered images into frequency sub-bands using the Discrete Wavelet Transform and calculates a weighted sum of squared errors between the rendered and ground-truth coefficients, with higher weights assigned to low-frequency sub-bands [S1, unofficial].

**Inputs and Outputs**
*   **Inputs**:
    *   $I$: Rendered image (RGB) from a batch of adjacently sampled rays.
    *   $I_{gt}$: Ground-truth image corresponding to the same camera pose.
    *   $\lambda_{sb}$: Weighting factors for each sub-band $sb \in \{LL, LH, HL, HH\}$.
    *   $K_{DW}$: Iteration interval for computing the loss.
    *   $T_{DW}$: Maximum iteration after which the loss is disabled.
*   **Outputs**:
    *   $L_{DW}$: Scalar loss value.

**Data Structures**
*   **Wavelet Filters**: 1D low-pass filter $\ell$ and high-pass filter $h$. Common types are Haar ($n=1$) or Daubechies ($dbN$, e.g., $db2, db3$) [S1, unofficial].
*   **2D Matrices**:
    *   $L$: Low-pass 2D matrix constructed by shifting rows of $\ell$.
    *   $H$: High-pass 2D matrix. If $\ell$ is orthogonal, $h$ is derived from $\ell$, and $H$ from $L$ [S1, unofficial].
*   **Sub-bands**: Four 2D arrays of size $\frac{H}{2} \times \frac{W}{2}$ (quarter the original image size):
    *   $I_{LL}$: Low frequencies in both directions.
    *   $I_{LH}$: Low horizontal, high vertical frequencies.
    *   $I_{HL}$: High horizontal, low vertical frequencies.
    *   $I_{HH}$: High frequencies in both directions [S1, unofficial].

**Numbered Steps**
1.  **Ray Sampling**: Sample rays in patches to form a 2D image $I$. This ensures rays are adjacent, forming a square grid compatible with 2D DWT [S1, unofficial]. Use random ray sampling for the standard MSE loss to capture global structure [S1, unofficial].
2.  **Filter Construction**: Construct the 2D low-pass matrix $L$ by shifting rows of the 1D low-pass wavelet filter $\ell$. Construct the high-pass matrix $H$ similarly using $\ell$ (if orthogonal) or derived $h$ [S1, unofficial].
3.  **DWT Decomposition**: Apply the DWT to $I$ and $I_{gt}$ using $L$ and $H$ to obtain the four sub-bands:
    $$ I_{LL} = L I L^T, \quad I_{LH} = H I L^T $$
    $$ I_{HL} = L I H^T, \quad I_{HH} = H I H^T $$
    (Note: The source notation $I_{LL} = L I L^T$ implies convolution/transform operations; standard DWT implementation involves filtering and downsampling) [S1, unofficial].
4.  **Loss Calculation**: Compute the squared Euclidean distance between corresponding sub-bands of $I$ and $I_{gt}$, weighted by $\lambda_{sb}$ [S1, unofficial].
5.  **Lazy Regularization**: Calculate $L_{DW}$ only every $K_{DW}$ iterations. Stop calculating it entirely after iteration $T_{DW}$ to facilitate high-frequency learning in later stages [S1, unofficial].
6.  **Combination**: Combine $L_{DW}$ with the photometric MSE loss $L_{MSE}$ and other regularizations [S1, unofficial].

**Formula**
The total Discrete Wavelet Loss is defined as [S1, unofficial]:

$$ L_{DW} = \sum_{sb \in \{LL,LH,HL,HH\}} \lambda_{sb} ||I_{sb} - I_{sb}^{gt}||^2_2 $$

Where:
*   $I_{sb}$ and $I_{sb}^{gt}$ are the sub-band images for the rendered and ground-truth views, respectively.
*   $||\cdot||^2_2$ denotes the squared L2 norm (sum of squared differences) over all pixels in the sub-band.
*   $\lambda_{sb}$ is the weight for sub-band $sb$.

*Note: The source formula in the notes included a term $\frac{\sum_{r \in R} ||c(r) - c_{gt}(r)||^2_2}{|R|}$, which is the definition of $L_{MSE}$, not $L_{DW}$. The formula above represents the specific DW component as described in the text [S1, unofficial].*

**Parameters**
*   **Sub-band Weights ($\lambda_{sb}$)**:
    *   $\lambda_{LL}$: Typically 0.4 for LLFF dataset, 0.04 for NeRF-Synthetic dataset [S1, unofficial].
    *   $\lambda_{LH}, \lambda_{HL}, \lambda_{HH}$: Typically 0.2 for LLFF dataset, 0.02 for NeRF-Synthetic dataset [S1, unofficial].
    *   $\lambda_{LL} > \lambda_{other}$ to prioritize low-frequency learning [S1, unofficial].
*   **Lazy Regularization**:
    *   $K_{DW}$: 10 iterations for LLFF, 150 iterations for NeRF-Synthetic [S1, unofficial].
    *   $T_{DW}$: 5000 iterations [S1, unofficial].
*   **Wavelet Type**: Haar ($n=1$) or Daubechies ($db2, db3$). Higher-order wavelets provide finer decomposition details but show no significant performance difference in final metrics [S1, unofficial].
*   **Patch Size**: $192 \times 192$ square patches (requiring $36864$ rays) [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_dwt_loss(rendered_image, gt_image, wavelet_filter_type='haar', 
                     lambda_ll=0.4, lambda_other=0.2, current_iter=0, 
                     t_dw=5000, k_dw=10):
    """
    Computes the Discrete Wavelet Transform Loss.
    
    Args:
        rendered_image (np.array): H x W x 3 array of rendered RGB values.
        gt_image (np.array): H x W x 3 array of ground-truth RGB values.
        wavelet_filter_type (str): 'haar' or 'db2'.
        lambda_ll (float): Weight for LL sub-band.
        lambda_other (float): Weight for LH, HL, HH sub-bands.
        current_iter (int): Current training iteration.
        t_dw (int): Maximum iteration to compute DW loss.
        k_dw (int): Iteration interval for DW loss.
        
    Returns:
        float: The DW loss value.
    """
    # Lazy Regularization: Skip if not at interval or past max iteration
    if current_iter % k_dw != 0 or current_iter > t_dw:
        return 0.0

    # 1. Construct Wavelet Matrices
    # L and H are 2D matrices constructed from 1D filters l and h.
    # For a 1D filter l of length N, L is a convolution matrix.
    # If orthogonal, h is derived from l.
    l, h = get_1d_wavelet_filters(wavelet_filter_type)
    L = construct_low_pass_matrix(l)
    H = construct_high_pass_matrix(h, L) # Derived if orthogonal

    # 2. Decompose Images into Sub-bands
    # Apply 2D DWT: Filter rows then columns, then downsample.
    # I_ll = L * I * L^T (conceptually, implemented via filtering/downsampling)
    # Note: Standard DWT implementation handles boundary conditions and downsampling.
    # The resulting sub-bands are half the spatial resolution of the input.
    
    # Assuming a standard DWT function that returns 4 sub-bands
    # I_ll, I_lh, I_hl, I_hh are H/2 x W/2 x 3 arrays
    I_ll, I_lh, I_hl, I_hh = discrete_wavelet_transform(rendered_image, L, H)
    gt_ll, gt_lh, gt_hl, gt_hh = discrete_wavelet_transform(gt_image, L, H)

    # 3. Calculate Weighted Squared Error
    # Compute L2 norm squared for each sub-band
    loss_ll = np.sum((I_ll - gt_ll) ** 2)
    loss_lh = np.sum((I_lh - gt_lh) ** 2)
    loss_hl = np.sum((I_hl - gt_hl) ** 2)
    loss_hh = np.sum((I_hh - gt_hh) ** 2)

    # Apply weights
    # lambda_ll is typically higher than lambda_other
    loss = (lambda_ll * loss_ll) + \
           (lambda_other * loss_lh) + \
           (lambda_other * loss_hl) + \
           (lambda_other * loss_hh)
           
    return loss

def get_1d_wavelet_filters(wavelet_type):
    """Returns 1D low-pass (l) and high-pass (h) filters."""
    if wavelet_type == 'haar':
        # Haar wavelet coefficients
        l = np.array([1, 1]) / np.sqrt(2)
        h = np.array([1, -1]) / np.sqrt(2)
    elif wavelet_type == 'db2':
        # Daubechies-2 coefficients (approximate)
        # Exact coefficients depend on normalization
        l = np.array([0.483, 0.836, 0.224, -0.129]) # Placeholder for exact db2
        h = np.array([-0.129, 0.224, 0.836, -0.483]) # Placeholder for exact db2
        # [G] Exact coefficients for db2: 
        # l = [0.4829629131445341, 0.836516303737469, 0.2241438680420134, -0.1294095225509214]
        # h = [-0.1294095225509214, -0.2241438680420134, 0.836516303737469, -0.4829629131445341]
        l = np.array([0.4829629131445341, 0.836516303737469, 0.2241438680420134, -0.1294095225509214])
        h = np.array([-0.1294095225509214, -0.2241438680420134, 0.836516303737469, -0.4829629131445341])
    else:
        raise ValueError("Unsupported wavelet type")
    return l, h

def construct_low_pass_matrix(l):
    """Constructs 2D low-pass matrix L from 1D filter l."""
    # L is a convolution matrix. For DWT, this is typically implemented as
    # filtering followed by downsampling.
    # Here we return the filter itself for use in a standard DWT function.
    return l

def construct_high_pass_matrix(h, L):
    """Constructs 2D high-pass matrix H."""
    # If orthogonal, H is derived from L.
    return h

def discrete_wavelet_transform(image, l, h):
    """
    Performs 2D DWT using filters l and h.
    Returns LL, LH, HL, HH sub-bands.
    """
    # 1. Filter rows
    # Convolve each row with l and h
    row_ll = convolve_1d(image, l, axis=1)
    row_lh = convolve_1d(image, h, axis=1)
    
    # 2. Downsample rows (keep every 2nd sample)
    row_ll_ds = row_ll[:, ::2, :]
    row_lh_ds = row_lh[:, ::2, :]
    
    # 3. Filter columns
    # Convolve each column of the downsampled results with l and h
    ll = convolve_1d(row_ll_ds, l, axis=0)
    lh = convolve_1d(row_lh_ds, l, axis=0)
    hl = convolve_1d(row_ll_ds, h, axis=0)
    hh = convolve_1d(row_lh_ds, h, axis=0)
    
    # 4. Downsample columns
    ll_ds = ll[::2, :, :]
    lh_ds = lh[::2, :, :]
    hl_ds = hl[::2, :, :]
    hh_ds = hh[::2, :, :]
    
    return ll_ds, lh_ds, hl_ds, hh_ds

def convolve_1d(data, kernel, axis):
    """1D convolution with 'same' padding."""
    # [G] Standard convolution implementation
    # For DWT, boundary handling is critical. 'Same' padding is typical.
    # This is a placeholder for the actual convolution logic.
    pass
```

**Edge Cases and Pitfalls**
*   **Sub-band Size**: Sub-bands are quarter the size of the original image. If the rendered image $I$ is too small, sub-bands become too coarse and visually meaningless [S1, unofficial]. This requires large images, incurring heavy computational overhead ($N^2$ rays for an $N$-sized square patch) [S1, unofficial].
*   **High-Frequency Content**: DWT high-frequency components include not just fine structural details but also specularity and light-dependent effects, which are not disentangled by the transform [S1, unofficial].
*   **Auxiliary Nature**: The DW loss does not eliminate high-frequency overfitting because it is auxiliary to the photometric MSE loss, which compares pixel-wise differences regardless of frequency [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: The DWT computation is $O(N^2)$ for an $N \times N$ image patch. Due to the large number of rays required for a meaningful patch ($192 \times 192 = 36864$ rays), it incurs significant overhead [S1, unofficial].
*   **Memory Complexity**: Requires storing the rendered image, ground-truth image, and intermediate sub-bands. The lazy regularization (computing every $K_{DW}$ iterations) mitigates memory and time costs [S1, unofficial].
*   **Performance**: On the 3-shot LLFF benchmark, DWTNeRF with DW loss/DB2 achieved PSNR 20.23, SSIM 0.678, LPIPS 0.193, compared to CombiNeRF baseline PSNR 20.12, SSIM 0.676, LPIPS 0.197 [S1, unofficial].
*   **Strengths**: Explicitly prioritizes low-frequency learning, which is crucial for few-shot NeRF convergence. Combines global (random ray sampling for MSE) and local (patch-based for DW) structure supervision [S1, unofficial].
*   **Limitations**: Does not disentangle high-frequency components like specularity. High-frequency overfitting may still occur due to the presence of MSE loss. Requires large patches for meaningful sub-bands, increasing computational cost [S1, unofficial].
*   **When to Use**: Few-shot NeRF training where low-frequency convergence is slow or unstable.
*   **When Not to Use**: When computational resources are limited and patch-based sampling is too expensive, or when high-frequency details (specularity) are the primary focus and need to be preserved without interference from low-frequency bias.

## Background
The Discrete Wavelet Transform Loss was introduced in the DWTNeRF paper to boost few-shot Neural Radiance Fields [S1, unofficial]. It builds on the CombiNeRF framework, which combines regularization and model-based approaches [S1, unofficial]. The method leverages the Discrete Wavelet Transform (DWT) to decompose images into frequency sub-bands, allowing for explicit supervision of low frequencies during early training stages [S1, unofficial]. The paper evaluates the method on the LLFF and NeRF-Synthetic datasets under few-shot conditions [S1, unofficial].

## Sources
[S1] DWTNeRF: Boosting Few-shot Neural Radiance Fields via Discrete Wavelet Transform, https://arxiv.org/abs/2501.12637v3, fetched 2026-10-27 (unofficial)
