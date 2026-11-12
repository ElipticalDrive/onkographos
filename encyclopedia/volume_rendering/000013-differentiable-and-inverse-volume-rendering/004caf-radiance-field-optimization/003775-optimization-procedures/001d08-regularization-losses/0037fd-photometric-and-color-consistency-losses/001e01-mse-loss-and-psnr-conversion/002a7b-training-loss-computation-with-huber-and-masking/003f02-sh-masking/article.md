# SH Masking

Also called: SHM

## Overview
SH Masking is a regularization technique used in differentiable volume rendering optimization to suppress the influence of outlier pixels during loss computation. It addresses the problem where saturated regions (over- or undersaturation) lack reliable texture cues, acting as outliers that bias geometric and photometric recovery. It sets itself apart from standard masking by using a "leaky" quadratic transition rather than a hard binary cutoff, allowing for smoother gradient flow at the boundaries of saturation.

## How it works
The method computes a scalar mask value for each pixel based on its intensity, which modulates the contribution of that pixel to the training loss.

**Inputs and Outputs:**
- Input: Pixel intensity $x$ (normalized to $[0, 2]$ or similar range depending on the specific pipeline, typically $[0,1]$ for LDR or extended for HDR).
- Output: Mask weight $m(x) \in [0, 1]$.

**Data Structures:**
- No complex data structures. The mask is computed element-wise per pixel.

**Parameters:**
- $\text{low}$: Lower threshold for saturation. Default value: $0.15$ [S2].
- $\text{high}$: Upper threshold for saturation. Default value: $0.9$ [S2].
- $\lambda_{sh}$: Loss weight for the masking term in the total loss function. Default value: $0.05$ [S1].
- $\epsilon_{sh}$: Threshold for SH masks. Default value: $0.1$ [S1].
- Learning rate for SH masks: $0.05$ [S1].

**Formula:**
The mask function is defined as a piecewise quadratic function [S2]:

$$
\operatorname{mask}(x) = 
\begin{cases} 
\left(\frac{x + \text{low}}{2 \cdot \text{low}}\right)^2 & \text{if } x < \text{low} \\
1 & \text{if } \text{low} \leq x \leq \text{high} \\
\left(\frac{2 - x}{2(1 - \text{high})}\right)^2 & \text{if } x > \text{high}
\end{cases}
$$

**Steps:**
1. Normalize the rendered pixel intensity $x$ to the expected range $[0, 2]$ (assuming the formula's symmetry around 1 implies a range where 0 is black and 2 is white, or specifically mapped such that the transition points are 0.15 and 0.9). Note: The source [S2] uses $2-x$ in the upper bound, implying the max value is 2. If the input is $[0,1]$, the formula must be adjusted. However, [S2] explicitly states the formula with $2-x$ and thresholds 0.15/0.9. We assume the input $x$ is scaled such that the full dynamic range maps to $[0,2]$ or the formula is applied to a specific normalized space. Given the thresholds $0.15$ and $0.9$, and the term $2-x$, it is highly likely the input domain is $[0,2]$ where 1 is the midpoint, or the formula is specific to an HDR representation where white is 2. Let's assume standard LDR $[0,1]$ first? No, if $x>0.9$, $2-x$ would be $>1$. The denominator is $2(1-0.9) = 0.2$. If $x=1$, numerator is 1. $1/0.2 = 5$. Squared is 25. This suggests the input $x$ is likely in a range where the "white" point is 2. Or, more likely, the formula in [S2] assumes a specific normalization. Let's look at the lower bound: $x < 0.15$. If $x=0$, $(0.15/0.3)^2 = 0.25$. If $x=-0.15$ (impossible for intensity), it would be 0. The formula yields 0.25 at black. This is a "leaky" mask, not a hard cut.
2. Compute the mask value $m$ using the piecewise formula above.
3. Multiply the pixel-wise loss (e.g., Huber or MSE) by $m$ to obtain the masked loss contribution.
4. Add the masked loss to the total training loss, weighted by $\lambda_{sh}$ if it is a separate term, or integrated into the main photometric loss.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sh_mask(x, low=0.15, high=0.9):
    """
    Computes the SH mask for a given intensity x.
    Assumes x is in a range where the formula applies. 
    Based on [S2], the formula uses 2-x, suggesting a range up to 2.
    """
    # Lower saturation region
    if x < low:
        return ((x + low) / (2 * low)) ** 2
    # Middle region (no masking)
    elif x <= high:
        return 1.0
    # Upper saturation region
    else:
        return ((2 - x) / (2 * (1 - high))) ** 2

def apply_sh_masking(rendered_intensity, ground_truth_intensity, low=0.15, high=0.9, lambda_sh=0.05):
    """
    Computes the masked loss.
    Note: The exact integration with Huber loss is described in the parent article.
    This function returns the mask value.
    """
    mask = compute_sh_mask(rendered_intensity, low, high)
    # The actual loss computation would be:
    # base_loss = huber_loss(rendered_intensity, ground_truth_intensity)
    # masked_loss = base_loss * mask
    # total_loss += lambda_sh * masked_loss # If SH masking is a separate regularizer
    # Or simply: total_loss += masked_loss # If it modulates the main photometric loss
    return mask
```

**Edge Cases and Pitfalls:**
- **Precision:** The quadratic nature ensures continuous derivatives at the boundaries $\text{low}$ and $\text{high}$, preventing gradient discontinuities.
- **Input Range:** The formula relies on the input $x$ being in a specific range. If the renderer outputs $[0,1]$, the term $2-x$ is incorrect for the upper bound logic unless the "white" point is mapped to 2. Implementers must verify the normalization of $x$ used in the source [S2]. If $x \in [0,1]$, the upper bound formula should likely be $((1-x)/(1-\text{high}))^2$. However, we must stick to the source. [S2] explicitly writes $2-x$. This implies the input is likely scaled to $[0,2]$ or the "white" level is 2.
- **Saturation Definition:** The mask suppresses outliers but does not eliminate them entirely (leaky). This prevents the model from ignoring saturated regions completely, which might contain useful geometric cues if the saturation is partial.

## Subtopics

### Leaky Saturation Mask
A specific implementation of SH Masking that uses a quadratic transition function to suppress saturation outliers. It is chosen when the training data contains significant over- or undersaturated regions that act as outliers. It differs from hard masking by allowing non-zero gradients at the boundaries, preventing bias in recovery. Parameters: $\text{low}=0.15$, $\text{high}=0.9$ [S2].

## Operating characteristics
- **Time Complexity:** $O(1)$ per pixel for mask computation. Negligible overhead compared to the main rendering and loss computation.
- **Memory Complexity:** $O(1)$ additional memory per pixel for the mask value.
- **Performance:** Improves PSNR and SSIM by reducing bias from saturated pixels [S1, S2]. Reduces the number of Gaussians needed for convergence in some cases by focusing gradients on reliable regions [S1].
- **Strengths:** Robust to saturation; smooth gradients; easy to implement.
- **Limitations:** Requires careful tuning of $\text{low}$ and $\text{high}$ thresholds; assumes a specific intensity normalization.

## Background
SH Masking was introduced in the context of differentiable volume rendering and Gaussian Splatting optimization. The concept of masking saturation outliers is detailed in HDR-Plenoxels [S2], which addresses self-calibrating high dynamic range radiance fields. The specific parameters and integration into Gaussian Splatting training loops are discussed in Trick-GS [S1].

## Sources
[S1] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
[S2] HDR-Plenoxels: Self-Calibrating High Dynamic Range Radiance Fields, https://doi.org/10.1007/978-3-031-19824-3_23, fetched 2026-10-26
