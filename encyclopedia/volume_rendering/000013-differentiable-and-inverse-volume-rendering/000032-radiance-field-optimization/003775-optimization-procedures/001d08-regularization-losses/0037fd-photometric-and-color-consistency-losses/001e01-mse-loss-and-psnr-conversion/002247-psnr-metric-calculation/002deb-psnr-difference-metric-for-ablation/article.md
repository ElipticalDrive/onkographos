# PSNR Difference Metric for Ablation
Also called: Delta PSNR

## Overview
The PSNR Difference Metric for Ablation quantifies the performance degradation caused by removing a specific component from a rendering pipeline by comparing the Peak Signal-to-Noise Ratio (PSNR) of the ablated configuration against the full method [S1, unofficial]. It isolates the error ratio independent of the bit depth (MAX), as MAX cancels out in the logarithmic difference, providing a scale-invariant measure of quality loss [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Inputs**: $\text{MSE}_{\text{full}}$ (Mean Squared Error of the full method), $\text{MSE}_{\text{config}}$ (Mean Squared Error of the ablated configuration).
- **Output**: $\Delta$ (The PSNR difference in decibels).

**Data Structures**
- No specific data structures are required; the metric operates on scalar MSE values derived from image comparisons.

**Steps**
1. Calculate the PSNR for the full method: $\text{PSNR}_{\text{full}} = 10 \log_{10} \frac{\text{MAX}^2}{\text{MSE}_{\text{full}}}$.
2. Calculate the PSNR for the ablated configuration: $\text{PSNR}_{\text{config}} = 10 \log_{10} \frac{\text{MAX}^2}{\text{MSE}_{\text{config}}}$.
3. Compute the difference $\Delta$ by subtracting the two PSNR values. The $\text{MAX}^2$ terms cancel out, simplifying the calculation to the ratio of the MSEs [S1, unofficial]:
   $$ \Delta = 10 \log_{10} \frac{\text{MSE}_{\text{full}}}{\text{MSE}_{\text{config}}} $$
4. To recover the error ratio from the difference, use the inverse relation [S1, unofficial]:
   $$ \frac{\text{MSE}_{\text{config}}}{\text{MSE}_{\text{full}}} = 10^{-\Delta/10} $$

**Parameters**
- **MAX**: The maximum possible pixel value (e.g., 255 for 8-bit images). This parameter is implicit in the cancellation and does not appear in the final difference formula [S1, unofficial].

**Edge Cases and Pitfalls**
- **Division by Zero**: If $\text{MSE}_{\text{config}}$ is zero, the logarithm is undefined. This implies the ablated method produces an identical result to the ground truth, which is rare in ablation studies involving component removal [G].
- **Sign Interpretation**: A positive $\Delta$ indicates the ablated method has higher error (worse quality) than the full method. A negative $\Delta$ indicates the ablated method has lower error (better quality), which may suggest the removed component was detrimental or the full method was overfitting [G].
- **Fixed Gaps**: A difference of 3 dB corresponds to exactly a factor of two in error, and 6 dB to a factor of four, independent of the absolute PSNR values [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def psnr_difference(mse_full: float, mse_config: float) -> float:
    """
    Calculate the PSNR difference between a full method and an ablated configuration.
    
    Args:
        mse_full: Mean Squared Error of the full method.
        mse_config: Mean Squared Error of the ablated configuration.
        
    Returns:
        Delta PSNR in decibels.
    """
    if mse_config == 0:
        raise ValueError("MSE of config cannot be zero (logarithm undefined).")
    if mse_full == 0:
        # If full MSE is 0, the full method is perfect. 
        # Any non-zero config MSE results in infinite loss.
        if mse_config > 0:
            return float('inf')
        return 0.0
        
    # Formula: Delta = 10 * log10(MSE_full / MSE_config)
    # This cancels out MAX^2 from the individual PSNR definitions.
    ratio = mse_full / mse_config
    delta = 10 * math.log10(ratio)
    return delta

def mse_ratio_from_delta(delta: float) -> float:
    """
    Recover the MSE ratio from the PSNR difference.
    
    Args:
        delta: The PSNR difference in decibels.
        
    Returns:
        The ratio MSE_config / MSE_full.
    """
    return 10 ** (-delta / 10)
```

## Subtopics

## Operating characteristics
- **Time Complexity**: $O(1)$ for the calculation itself, assuming MSE values are precomputed.
- **Memory Complexity**: $O(1)$.
- **Strengths**: It provides a single scalar value that facilitates objective comparison of compression techniques or architectural changes, avoiding the impracticality of displaying full rate-distortion curves [S1, unofficial]. It is independent of the bit depth of the image format [S1, unofficial].
- **Limitations**: It only measures the difference relative to a specific baseline (the full method) and does not provide an absolute quality metric [G]. It assumes the MSE values are computed on the same pixel grid and color space [G].

## Background
The metric is derived from the properties of the logarithmic scale used in PSNR calculations. It is used in ablation studies to isolate the error ratio caused by removing a specific component from a rendering pipeline, such as in 3D Gaussian Splatting [S1, unofficial]. The cancellation of the MAX term is a key property that makes the metric robust across different image formats [S1, unofficial].

## Sources
[S1] An Architectural Review of the 3D Gaussian Splatting Rendering Pipeline, https://doi.org/10.5281/zenodo.21739653, fetched 2026-10-26 (unofficial)
