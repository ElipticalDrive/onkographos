# Discrete Wavelet Transform Loss

Also called: DW loss, Discrete Wavelet Loss, LDW

## Overview
The Discrete Wavelet Transform (DWT) Loss is a regularization term for Neural Radiance Fields (NeRF) that minimizes the difference between the frequency representations of rendered and ground-truth images by decomposing them into sub-bands [S1, unofficial]. It addresses high-frequency overfitting in few-shot NeRF training by explicitly prioritizing low-frequency learning during early stages [S1, unofficial]. Unlike its sibling [MSE Loss and PSNR Conversion](1651-mse-loss-and-psnr-conversion.md), which compares pixel-wise values globally, the DW loss operates on local structures using patch-based sampling to supervise low and mid-frequency components [S1, unofficial].

## How it works
The method decomposes rendered images into frequency sub-bands using the Discrete Wavelet Transform and calculates a weighted sum of squared errors between the rendered and ground-truth coefficients, with higher weights assigned to low-frequency sub-bands [S1, unofficial].

**Inputs and Outputs**
*   **Inputs**:
    *   $I$: Rendered image (RGB) from a batch of adjacently sampled rays [S1, unofficial].
    *   $I_{gt}$: Ground-truth image corresponding to the same camera pose [S1, unofficial].
    *   $\lambda_{sb}$: Weighting factors for each sub-band $sb \in \{LL, LH, HL, HH\}$ [S1, unofficial].
    *   $K_{DW}$: Iteration interval for computing the loss [S1, unofficial].
    *   $T_{DW}$: Maximum iteration after which the loss is disabled [S1, unofficial].
*   **Outputs**:
    *   $L_{DW}$: Scalar loss value [S1, unofficial].

**Data Structures**
*   **Wavelet Filters**: 1D low-pass filter $\ell$ and high-pass filter $h$ [S1, unofficial]. Common types are Haar ($n=1$) or Daubechies ($dbN$, e.g., $db2, db3$) [S1, unofficial].
*   **2D Matrices**:
    *   $L$: Low-pass 2D matrix constructed by shifting rows of $\ell$ [S1, unofficial].
    *   $H$: High-pass 2D matrix [S1, unofficial]. If $\ell$ is orthogonal, $h$ is derived from $\ell$, and $H$ is derived from $L$ [S1, unofficial].
*   **Sub-bands**: Four 2D arrays of size $\frac{H}{2} \times \frac{W}{2}$ (quarter the size of the original image) [S1, unofficial]:
    *   $I_{LL}$: Low frequencies in both directions [S1, unofficial].
    *   $I_{LH}$: Low horizontal, high vertical frequencies [S1, unofficial].
    *   $I_{HL}$: High horizontal, low vertical frequencies [S1, unofficial].
    *   $I_{HH}$: High frequencies in both directions [S1, unofficial].

**Numbered Steps**
1.  **Ray Sampling**: Sample rays in patches to form a 2D image $I$ [S1, unofficial]. This ensures rays are adjacent, forming a square grid compatible with 2D DWT [S1, unofficial]. Use random ray sampling for the standard MSE loss to capture global structure [S1, unofficial].
2.  **Filter Construction**: Construct the 2D low-pass matrix $L$ by shifting rows of the 1D low-pass wavelet filter $\ell$ [S1, unofficial]. Construct the high-pass matrix $H$ similarly using $\ell$ (if orthogonal) or derived $h$ [S1, unofficial].
3.  **DWT Decomposition**: Apply the DWT to $I$ and $I_{gt}$ using $L$ and $H$ to obtain the four sub-bands [S1, unofficial]:
    $$ I_{LL} = L I L^T, \quad I_{LH} = H I L^T $$
    $$ I_{HL} = L I H^T, \quad I_{HH} = H I H^T $$
    (Note: The source notation $I_{LL} = L I L^T$ implies convolution/transform operations; standard DWT implementation involves filtering and downsampling) [S1, unofficial].
4.  **Loss Calculation**: Compute the squared Euclidean distance between corresponding sub-bands of $I$ and $I_{gt}$, weighted by $\lambda_{sb}$ [S1, unofficial].
5.  **Lazy Regularization**: Calculate $L_{DW}$ only every $K_{DW}$ iterations [S1, unofficial]. Stop calculating it entirely after iteration $T_{DW}$ to facilitate high-frequency learning in later stages [S1, unofficial].
6.  **Combination**: Combine $L_{DW}$ with the photometric MSE loss $L_{MSE}$ and other regularizations [S1, unofficial].

**Formula**
The total Discrete Wavelet Loss is defined as [S1, unofficial]:

$$ L_{DW} = \sum_{sb \in \{LL,LH,HL,HH\}} \lambda_{sb} ||I_{sb} - I_{sb}^{gt}||^2_2 $$

Where:
*   $I_{sb}$ and $I_{sb}^{gt}$ are the sub-band images for the rendered and ground-truth views, respectively [S1, unofficial].
*   $||\cdot||^2_2$ denotes the squared L2 norm (sum of squared differences) over all pixels in the sub-band [S1, unofficial].
*   $\lambda_{sb}$ is the weight for sub-band $sb$ [S1, unofficial].

The source notes initially included a term $\frac{\sum_{r \in R} ||c(r) - c_{gt}(r)||^2_2}{|R|}$, but this is the definition of $L_{MSE}$, not $L_{DW}$ [S1, unofficial]. The formula above represents the specific DW component as described in the text [S1, unofficial].

**Parameters**
*   **Sub-band Weights ($\lambda_{sb}$)**:
    *   $\lambda_{LL}$: Typically 0.4 for LLFF dataset, 0.04 for NeRF-Synthetic dataset [S1, unofficial].
    *   $\lambda_{LH}, \lambda_{HL}, \lambda_{HH}$: Typically 0.2 for LLFF dataset, 0.02 for NeRF-Synthetic dataset [S1, unofficial].
    *   $\lambda_{LL} > \lambda_{other}$ to prioritize low-frequency learning [S1, unofficial].
*   **Lazy Regularization**:
    *   $K_{DW}$: 10 iterations for LLFF, 150 iterations for NeRF-Synthetic [S1, unofficial].
    *   $T_{DW}$: 5000 iterations [S1, unofficial].
*   **Wavelet Type**: Haar ($n=1$) or Daubechies ($db2, db3$) [S1, unofficial]. Higher-order wavelets provide finer decomposition details but show no significant performance difference in final metrics [S1, unofficial].
*   **Patch Size**: $192 \times 192$ square patches (requiring $36864$ rays) [S1, unofficial].
*   **Sub-band Size**: Sub-bands are quarter the size of the original image [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_dwt_loss(rendered_image, gt_image, wavelet_type='haar', 
                     lambda_ll=0.4, lambda_other=0.2, current_iter=0, 
                     t_dw=5000, k_dw=10):
    # Lazy Regularization: Skip if not at interval or past max iteration
    if current_iter % k_dw != 0 or current_iter > t_dw:
        return 0.0

    # 1. Get 1D wavelet filters
    if wavelet_type == 'haar':
        l = [1.0 / 2**0.5, 1.0 / 2**0.5]
        h = [1.0 / 2**0.5, -1.0 / 2**0.5]
    elif wavelet_type == 'db2':
        l = [0.4829629131445341, 0.836516303737469, 0.2241438680420134, -0.1294095225509214]
        h = [-0.1294095225509214, -0.2241438680420134, 0.836516303737469, -0.4829629131445341]
    else:
        raise ValueError("Unsupported wavelet type")

    # 2. Decompose images into sub-bands
    I_ll, I_lh, I_hl, I_hh = apply_2d_dwt(rendered_image, l, h)
    gt_ll, gt_lh, gt_hl, gt_hh = apply_2d_dwt(gt_image, l, h)

    # 3. Calculate weighted squared error
    loss_ll = sum((I_ll[r][c][ch] - gt_ll[r][c][ch])**2 for r in range(len(I_ll)) for c in range(len(I_ll[0])) for ch in range(3))
    loss_lh = sum((I_lh[r][c][ch] - gt_lh[r][c][ch])**2 for r in range(len(I_lh)) for c in range(len(I_lh[0])) for ch in range(3))
    loss_hl = sum((I_hl[r][c][ch] - gt_hl[r][c][ch])**2 for r in range(len(I_hl)) for c in range(len(I_hl[0])) for ch in range(3))
    loss_hh = sum((I_hh[r][c][ch] - gt_hh[r][c][ch])**2 for r in range(len(I_hh)) for c in range(len(I_hh[0])) for ch in range(3))

    loss = (lambda_ll * loss_ll) + (lambda_other * loss_lh) + (lambda_other * loss_hl) + (lambda_other * loss_hh)
    return loss

def apply_2d_dwt(image, l, h):
    # image is H x W x 3
    H, W, C = len(image), len(image[0]), len(image[0][0])
    
    # Filter rows
    row_ll = convolve_1d(image, l, axis=1)
    row_lh = convolve_1d(image, h, axis=1)
    
    # Downsample rows
    row_ll_ds = row_ll[::2, :, :]
    row_lh_ds = row_lh[::2, :, :]
    
    # Filter columns
    ll = convolve_1d(row_ll_ds, l, axis=0)
    lh = convolve_1d(row_lh_ds, l, axis=0)
    hl = convolve_1d(row_ll_ds, h, axis=0)
    hh = convolve_1d(row_lh_ds, h, axis=0)
    
    # Downsample columns
    ll_ds = ll[::2, :, :]
    lh_ds = lh[::2, :, :]
    hl_ds = hl[::2, :, :]
    hh_ds = hh[::2, :, :]
    
    return ll_ds, lh_ds, hl_ds, hh_ds

def convolve_1d(data, kernel, axis):
    # 1D convolution with zero-padding
    # axis=0: rows, axis=1: columns
    if axis == 1:
        # Convolve along columns
        H, W, C = len(data), len(data[0]), len(data[0][0])
        out = [[[0.0]*C for _ in range(W)] for _ in range(H)]
        k_len = len(kernel)
        pad = k_len // 2
        for r in range(H):
            for c in range(W):
                for ch in range(C):
                    val = 0.0
                    for k in range(k_len):
                        idx = c - pad + k
                        if 0 <= idx < W:
                            val += data[r][idx][ch] * kernel[k]
                        else:
                            pass # [G] Zero-padding at boundaries
                    out[r][c][ch] = val
        return out
    else:
        # Convolve along rows
        H, W, C = len(data), len(data[0]), len(data[0][0])
        out = [[[0.0]*C for _ in range(W)] for _ in range(H)]
        k_len = len(kernel)
        pad = k_len // 2
        for r in range(H):
            for c in range(W):
                for ch in range(C):
                    val = 0.0
                    for k in range(k_len):
                        idx = r - pad + k
                        if 0 <= idx < H:
                            val += data[idx][c][ch] * kernel[k]
                        else:
                            pass # [G] Zero-padding at boundaries
                    out[r][c][ch] = val
        return out
```

**Edge Cases and Pitfalls**
*   **Sub-band Size**: Sub-bands are quarter the size of the original image [S1, unofficial]. If the rendered image $I$ is too small, sub-bands become too coarse and visually meaningless [S1, unofficial]. This requires large images, incurring heavy computational overhead ($N^2$ rays for an $N$-sized square patch) [S1, unofficial].
*   **High-Frequency Content**: DWT high-frequency components include not just fine structural details but also specularity and light-dependent effects, which are not disentangled by the transform [S1, unofficial].
*   **Auxiliary Nature**: The DW loss does not eliminate high-frequency overfitting because it is auxiliary to the photometric MSE loss, which compares pixel-wise differences regardless of frequency [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: The DWT computation is $O(N^2)$ for an $N \times N$ image patch [S1, unofficial]. Due to the large number of rays required for a meaningful patch ($192 \times 192 = 36864$ rays), it incurs significant overhead [S1, unofficial].
*   **Memory Complexity**: Requires storing the rendered image, ground-truth image, and intermediate sub-bands [S1, unofficial]. The lazy regularization (computing every $K_{DW}$ iterations) mitigates memory and time costs [S1, unofficial].
*   **Performance**: On the 3-shot LLFF benchmark, DWTNeRF with DW loss/DB2 achieved PSNR 20.23, SSIM 0.678, LPIPS 0.193, compared to CombiNeRF baseline PSNR 20.12, SSIM 0.676, LPIPS 0.197 [S1, unofficial].
*   **Strengths**: Explicitly prioritizes low-frequency learning, which is crucial for few-shot NeRF convergence [S1, unofficial]. Combines global (random ray sampling for MSE) and local (patch-based for DW) structure supervision [S1, unofficial].
*   **Limitations**: Does not disentangle high-frequency components like specularity [S1, unofficial]. High-frequency overfitting may still occur due to the presence of MSE loss [S1, unofficial]. Sub-bands are quarter the size of the original image; if the rendered image is too small, sub-bands become too coarse, requiring large images that incur heavy computational overhead ($N^2$ rays for an $N$-sized square patch) [S1, unofficial].
*   **When to Use**: Few-shot NeRF training where low-frequency convergence is slow or unstable [S1, unofficial].
*   **When Not to Use**: When computational resources are limited and patch-based sampling is too expensive, or when high-frequency details (specularity) are the primary focus and need to be preserved without interference from low-frequency bias [S1, unofficial].

## Background
The Discrete Wavelet Transform Loss was introduced in the DWTNeRF paper to boost few-shot Neural Radiance Fields [S1, unofficial]. It builds on the CombiNeRF framework, which combines regularization and model-based approaches [S1, unofficial]. The method leverages the Discrete Wavelet Transform (DWT) to decompose images into frequency sub-bands, allowing for explicit supervision of low frequencies during early training stages [S1, unofficial]. The paper evaluates the method on the LLFF and NeRF-Synthetic datasets under few-shot conditions [S1, unofficial].

## Sources
[S1] DWTNeRF: Boosting Few-shot Neural Radiance Fields via Discrete Wavelet Transform, https://arxiv.org/abs/2501.12637v3, fetched 2026-10-27 (unofficial)
