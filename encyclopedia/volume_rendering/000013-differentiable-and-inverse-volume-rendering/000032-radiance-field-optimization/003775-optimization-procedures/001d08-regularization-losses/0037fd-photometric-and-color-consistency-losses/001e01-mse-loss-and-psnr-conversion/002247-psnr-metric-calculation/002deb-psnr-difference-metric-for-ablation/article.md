# PSNR Difference Metric for Ablation

Also called: Delta PSNR

## Overview
The PSNR Difference Metric for Ablation quantifies the performance degradation caused by removing a specific component from a rendering pipeline by comparing the Peak Signal-to-Noise Ratio (PSNR) of a reduced configuration against a full baseline. It isolates the error ratio independent of the bit depth (MAX), as MAX cancels out in the logarithmic difference [S1, unofficial]. This metric sets itself apart from absolute metrics like [PSNR Metric Calculation](2742-psnr-metric-calculation.md) by providing a relative measure of loss, and from [3D Evaluation Metrics](2903-3d-evaluation-metrics.md) by focusing specifically on the delta between two specific configurations rather than a global scene score.

## How it works
**Inputs and Outputs:**
- **Inputs:** $\text{MSE}_{\text{config}}$ (Mean Squared Error of the ablated configuration), $\text{MSE}_{\text{full}}$ (Mean Squared Error of the full baseline method).
- **Output:** $\Delta$ (The PSNR difference in dB).

**Data Structures:**
No complex data structures are required; the metric operates on scalar MSE values derived from pixel-wise comparisons.

**Steps:**
1. Calculate the PSNR for the full method: $\text{PSNR}_{\text{full}} = 10 \log_{10} \frac{\text{MAX}^2}{\text{MSE}_{\text{full}}}$.
2. Calculate the PSNR for the ablated configuration: $\text{PSNR}_{\text{config}} = 10 \log_{10} \frac{\text{MAX}^2}{\text{MSE}_{\text{config}}}$.
3. Compute the difference $\Delta$ by subtracting the two PSNR values. The $\text{MAX}^2$ terms cancel out, simplifying the calculation to the ratio of the MSEs [S1, unofficial]:
   $$ \Delta = \text{PSNR}_{\text{config}} - \text{PSNR}_{\text{full}} = 10 \log_{10} \frac{\text{MSE}_{\text{full}}}{\text{MSE}_{\text{config}}} $$
4. To recover the error ratio from the difference, use the inverse relation [S1, unofficial]:
   $$ \frac{\text{MSE}_{\text{config}}}{\text{MSE}_{\text{full}}} = 10^{-\Delta/10} $$

**Parameters:**
- **MAX:** The maximum possible pixel value (e.g., 255 for 8-bit). It is not needed for the final calculation of $\Delta$ as it cancels out, provided both MSEs are computed using the same bit depth [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** The metric relies on the stability of the MSE calculation. If $\text{MSE}_{\text{config}}$ is zero (perfect reconstruction), the logarithm is undefined.
- **Interpretation:** A fixed gap in $\Delta$ corresponds to a fixed ratio of error, not a fixed amount. For example, a difference of 3 dB corresponds to exactly a factor of two in error ($10^{3/10} \approx 2$), and 6 dB corresponds to a factor of four, independent of the absolute PSNR values [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def calculate_psnr_difference(mse_full: float, mse_config: float) -> float:
    """
    Calculates the PSNR difference (Delta PSNR) between a full method and an ablated configuration.
    
    Args:
        mse_full: Mean Squared Error of the full baseline method.
        mse_config: Mean Squared Error of the ablated configuration.
        
    Returns:
        Delta PSNR in dB.
    """
    if mse_full <= 0 or mse_config <= 0:
        raise ValueError("MSE values must be positive.")
    
    # Formula: Delta = 10 * log10(MSE_full / MSE_config)
    # The MAX^2 term cancels out in the subtraction of two PSNRs.
    ratio = mse_full / mse_config
    delta_psnr = 10 * math.log10(ratio)
    
    return delta_psnr

def recover_error_ratio(delta_psnr: float) -> float:
    """
    Recovers the ratio of MSE_config / MSE_full from the Delta PSNR.
    
    Args:
        delta_psnr: The PSNR difference in dB.
        
    Returns:
        The ratio MSE_config / MSE_full.
    """
    # Formula: MSE_config / MSE_full = 10^(-Delta / 10)
    return math.pow(10, -delta_psnr / 10)
```

## Subtopics

### Bjøntegaard Delta PSNR (BD-PSNR) and BD-BR
BD-PSNR and BD-BR are metrics used to evaluate lossy compression performance by calculating the average difference in quality (BD-PSNR) or bit-rate (BD-BR) between two rate-distortion curves [S2]. Originally designed within the Video Coding Experts Group (VCEG) – ITU SG 16 – for comparing video coding techniques, BD-PSNR measures the difference in compression performance by calculating the average difference in quality between two rate-distortion curves, represented by the area between the curves [S2]. The result is a single PSNR value (in dB) that facilitates the objective comparison of investigated compression techniques. Similarly, BD-BR calculates the average bit-rate difference. Implementation for these metrics has been written in Excel VBA [S2].

## Operating characteristics
- **Time Complexity:** $O(1)$ relative to the number of pixels, as it operates on pre-computed scalar MSE values. The calculation of the MSE itself is $O(N)$ where $N$ is the number of pixels.
- **Memory Complexity:** $O(1)$ for the metric calculation itself.
- **Strengths:** It provides a bit-depth-independent comparison of error ratios. A fixed dB difference always implies a fixed error ratio (e.g., 3 dB = 2x error), making it intuitive for ablation studies [S1, unofficial].
- **Limitations:** It only compares two specific configurations. It does not account for perceptual quality beyond MSE. It is sensitive to the choice of the baseline "full" method.
- **When to use:** When performing ablation studies to quantify the contribution of specific components to the overall rendering quality.
- **When not to use:** When comparing methods with different bit depths or when perceptual metrics are more appropriate than MSE-based metrics.

## Background
The PSNR Difference Metric for Ablation is derived from the properties of the logarithmic scale used in PSNR calculations. The key property that the MAX term cancels out when subtracting two PSNR values was highlighted in architectural reviews of 3D Gaussian Splatting rendering pipelines [S1, unofficial]. This allows for a clean comparison of error ratios without needing to specify the bit depth explicitly in the delta calculation. The metric is particularly useful in the context of differentiable volume rendering and radiance-field optimization, where ablation studies are common to understand the impact of various optimization procedures and regularization losses.

## Sources
[S1] An Architectural Review of the 3D Gaussian Splatting Rendering Pipeline, https://doi.org/10.5281/zenodo.21739653, fetched 2026-10-26 (unofficial)
[S2] Wavelet based volumetric medical image compression, https://doi.org/10.1016/j.image.2014.12.007, fetched 2026-10-26
