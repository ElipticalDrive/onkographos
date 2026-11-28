# Intensity-to-Grayscale Mapping

Also called: Intensity Normalization and Gamma Correction, Percentile Normalization with Gamma Compression, luminance mapping

## Overview
Intensity-to-Grayscale Mapping converts absolute scalar field intensity values into 8-bit grayscale luminance indices in the range $[0, 255]$ [S2, unofficial]. It solves the problem of visualizing high-precision floating-point data by compressing the dynamic range and enhancing contrast in low-intensity regions through gamma compression [S2, unofficial]. Unlike simple linear normalization, this method uses percentile-based saturation thresholds to ignore outliers and applies a non-linear power law to brighten subtle features [S2, unofficial].

## How it works
**Inputs:**
- `intensity`: A scalar field of floating-point values.
- `gamma`: A float compression exponent. Default is $0.55$ [S2, unofficial].
- `percentile`: The percentile threshold for saturation (default $99$) [S2, unofficial].

**Outputs:**
- `result`: An array of integers in $[0, 255]$ representing grayscale luminance.

**Steps:**
1. **Filter Finite Values:** Identify all finite values in the input intensity field to prevent calculation errors [S2, unofficial].
2. **Calculate Saturation Threshold ($b_{\text{max}}$):** Compute the $99$-th percentile of the finite intensity values and add a small epsilon ($10^{-9}$) to ensure the denominator is non-zero [S2, unofficial].
   $$ b_{\text{max}} = \text{percentile}(\text{finite\_intensity}, 99) + 10^{-9} $$
3. **Normalize:** Map the raw intensity to $[0, 1]$ using the saturation threshold [S2, unofficial]. Values below $0$ are clamped to $0$, and values above $b_{\text{max}}$ are clamped to $1$ [S2, unofficial].
   $$ \text{norm} = \max(0.0, \min(1.0, \frac{\text{intensity}}{b_{\text{max}}})) $$
   *Note: An alternative implementation uses a global min/max range instead of a percentile threshold, computing $\max(0.0, \min(1.0, (\text{intensity} - i_{\text{min}}) / \max(i_{\text{max}} - i_{\text{min}}, 10^{-9})))$ [S2, unofficial].*
4. **Gamma Compression:** Apply the gamma power function to the normalized value [S2, unofficial].
   $$ \text{compressed} = \text{norm}^{\gamma} $$
5. **Scale and Quantize:** Multiply by $255$ and round to the nearest integer to produce the final 8-bit index [S2, unofficial].
   $$ \text{result} = \text{int}(\text{round}(\text{compressed} \times 255)) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def intensity_to_grayscale(intensity: list[float], gamma: float = 0.55, percentile: float = 99.0) -> list[int]:
    """
    Map intensity values to 8-bit grayscale using percentile normalization and gamma compression.
    """
    # Step 1: Filter finite values
    finite = [v for v in intensity if v == v]  # v != v detects NaN
    
    if not finite:
        return [0] * len(intensity)

    # Step 2: Calculate saturation threshold b_max
    finite.sort()
    n = len(finite)
    # Linear interpolation for percentile (matches np.percentile default)
    idx = (percentile / 100.0) * (n - 1)
    lower = int(idx)
    upper = lower + 1
    frac = idx - lower
    if upper >= n:
        b_max = finite[-1] + 1e-9
    else:
        b_max = finite[lower] + frac * (finite[upper] - finite[lower]) + 1e-9

    # Steps 3, 4, 5: Normalize, Gamma, Scale
    result = []
    for v in intensity:
        # Step 3: Normalize to [0, 1]
        norm = max(0.0, min(1.0, v / b_max))
        
        # Step 4: Gamma compression
        compressed = norm ** gamma
        
        # Step 5: Scale to 8-bit
        val = int(round(compressed * 255))
        
        # Clamp final result to [0, 255]
        result.append(max(0, min(255, val)))
        
    return result
```

**Parameters:**
- `gamma`: Compression exponent. Values $< 1$ expand the mid-range and brighten low-end features [S2, unofficial]. Default is $0.55$ for Earth models, $0.7$ for Saturn models [S2, unofficial].
- `percentile`: The percentile used to determine the upper bound of the normalization range. Default is $99$ [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** If the global range is negligible ($i_{\text{max}} \approx i_{\text{min}}$), the denominator is clamped to $10^{-9}$ [S2, unofficial]. If the range is less than $10^{-6}$, return $0.0$ to avoid division by zero [S1, unofficial].
- **NaN/Inf:** Non-finite values must be filtered or clamped before normalization to prevent propagation of NaNs [S2, unofficial].
- **Clamping:** Normalized values must be clamped to $[0.0, 1.0]$ before applying gamma to ensure valid output [S2, unofficial].

## Subtopics

### [Weighted Percentile Computation](7031-weighted-percentile-computation.md)
Computes percentiles when data points have associated weights, useful for non-uniform sampling. Choose when data density varies significantly across the volume. [G]

## Operating characteristics
- **Time Complexity:** [G] Not given in the sources.
- **Memory Complexity:** [G] Not given in the sources.
- **Performance:** [G] Not given in the sources.
- **Strengths:** [G] Not given in the sources.
- **Limitations:** [G] Not given in the sources.

## Background
The method is used in astrophysical vector field visualization, specifically in the AstroSplat-Browser project for visualizing magnetospheric data (Earth and Saturn models) [S2, unofficial]. It builds on percentile-based normalization techniques to handle the wide dynamic range of physical field intensities [S2, unofficial].

## Sources
[S1] xwghua/clearvolume-plus, https://github.com/xwghua/clearvolume-plus, fetched 2026-10-26 (unofficial)
[S2] VitorFigueiredoResearch/AstroSplat-Browser-native-volumetric-visualization-for-astrophysical-vector-fields, https://github.com/VitorFigueiredoResearch/AstroSplat-Browser-native-volumetric-visualization-for-astrophysical-vector-fields, fetched 2026-10-26 (unofficial)
