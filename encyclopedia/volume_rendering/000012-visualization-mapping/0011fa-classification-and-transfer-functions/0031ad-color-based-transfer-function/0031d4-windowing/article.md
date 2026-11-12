# Windowing

Also called: Window Specification, Window/Level

## Overview
Windowing is a scalar-to-color mapping technique that restricts the visualization to a specific range of data values, mapping those values to a linear grayscale ramp while clamping out-of-range values to black or white [S1, unofficial]. It solves the problem of low contrast in volumetric data by isolating a "window" of interest (e.g., soft tissue) from the full data range, preventing background noise or irrelevant high-intensity structures (e.g., bone) from obscuring the target features [S1, unofficial]. Unlike [Colormap Application for Visualization](7714-colormap-application-for-visualization.md), which maps the entire domain to a spectrum, or [Clamping](11530-clamping.md), which maps out-of-range values to the boundary colors of the transfer function, windowing explicitly discards out-of-range information into neutral background colors [S1, unofficial][S2].

## How it works
**Inputs and Outputs**
*   Input: A scalar data value $x$.
*   Parameters: Window center $c$ (the midpoint of the range), window width $w$ (where $w \ge 1$).
*   Output: A color value $C$ in the range $[0, 1]$.

**Data Structures**
No complex data structures are required. The method relies on two scalar parameters, $c$ and $w$, and the input scalar $x$.

**Steps and Formula**
The mapping is defined by a piecewise linear function. The lower bound of the window is $L = c - 0.5 - (w-1)/2$ and the upper bound is $R = c - 0.5 + (w-1)/2$.

1.  If $x \le L$, set $C = 0$ (black).
2.  Else if $x > R$, set $C = 1$ (white).
3.  Else, calculate $C$ using the linear interpolation:
    $$ C = \frac{x - (c - 0.5)}{w - 1} + 0.5 $$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def windowing(x: float, c: float, w: float) -> float:
    """
    Maps scalar value x to grayscale C using window center c and width w.
    """
    # Calculate bounds based on the definition in [S1, unofficial]
    # Lower bound: c - 0.5 - (w-1)/2
    # Upper bound: c - 0.5 + (w-1)/2
    
    lower_bound = c - 0.5 - (w - 1) / 2
    upper_bound = c - 0.5 + (w - 1) / 2
    
    if x <= lower_bound:
        return 0.0
    elif x > upper_bound:
        return 1.0
    else:
        # Linear ramp from 0 to 1 across the window width
        # Note: The formula in [S1, unofficial] adds 0.5 to center the ramp [0,1]
        # around the visual center, but effectively maps the window
        # to the full [0,1] intensity range.
        return (x - (c - 0.5)) / (w - 1) + 0.5
```

**Parameters**
*   $c$: Center of the window. Typical values depend on the data type (e.g., Hounsfield units in CT).
*   $w$: Width of the window. Must be $\ge 1$.
*   $x$: Input data value.

**Edge Cases and Pitfalls**
*   **Division by Zero:** If $w=1$, the denominator $(w-1)$ becomes zero. The source [S1, unofficial] specifies $w \ge 1$, but the formula requires $w > 1$ for the linear interpolation step. If $w=1$, the window has zero width, and only the single value $x = c - 0.5$ should theoretically map to the center, but the linear ramp is undefined. In practice, $w$ should be strictly greater than 1 for this specific formula [S1, unofficial].
*   **Precision:** Floating-point comparisons at the boundaries $L$ and $R$ must be consistent with the `<=` and `>` operators to avoid gaps or overlaps in the mapping.

## Subtopics

### [Channel Window and Transfer Function Initialization](8807-channel-window-and-transfer-function-initialization.md)
This method initializes transfer function parameters for multi-material volume rendering by mapping specific data value ranges to opacity profiles for distinct materials (background, tumor, vessel, bone) [8807-channel-window-and-transfer-function-initialization.md]. It differs from generic windowing by hardcoding material-specific opacity ramps based on fixed sample value thresholds (e.g., 122–142) rather than deriving them dynamically from user-defined color maps, ensuring consistent segmentation of these four specific tissue types [8807-channel-window-and-transfer-function-initialization.md].

### [Clamping](11530-clamping.md)
Clamping is a boundary handling technique used within transfer functions to define the behavior of scalar values that fall outside the defined data range [11530-clamping.md]. While [Windowing](6611-windowing.md) maps out-of-range values to black or white, clamping maps them to the boundary colors of the transfer function itself [11530-clamping.md]. This prevents the loss of information for extreme values by preserving their association with the nearest defined color rather than discarding them into a neutral background [11530-clamping.md].

### Narrow Interval Weighting
Narrow Interval Weighting is a weighting function used in level-of-detail selection that is 1 for values in a specific narrow range and 0 elsewhere, focusing error reduction on salient intervals [S2]. It differs from wide-interval weighting (uniformly 1) by covering only the minimal interval widths needed to cover non-transparent portions of color-opacity transfer functions, yielding results closer to ground truth [S2]. A key pitfall is that artifacts typical of block-wise downsampling, including smoothing effects and discontinuities at block boundaries, become more pronounced as the interval width increases [S2]. For the QVAPOR variable in climate test data, RMS error increases as the interval width increases from 0.1 to 0.9 [S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample, as it involves simple arithmetic comparisons and linear interpolation.
*   **Memory Complexity:** $O(1)$, requiring only two scalar parameters ($c, w$).
*   **Strengths:** Extremely fast; effectively isolates features of interest by suppressing irrelevant data ranges; simple to implement and tune [S1, unofficial].
*   **Limitations:** Discards all information outside the window (mapped to black/white); sensitive to the choice of $c$ and $w$; does not handle multi-material segmentation natively without multiple windows [S1, unofficial].
*   **When to use:** When the user knows the approximate data range of the target structure and wants to maximize contrast for that range [S1, unofficial].
*   **When not to use:** When preserving the full dynamic range or visualizing multiple distinct materials with different intensity ranges simultaneously without complex multi-window setups [S1, unofficial].

## Background
Windowing is one of the simplest transfer functions, introduced in early volume visualization literature as a method to map a range of data values ("window") to a linear ramp of grayscales [S1, unofficial]. It is commonly defined by its center $c$ and width $w$ [S1, unofficial]. Histograms of the data values are often used to guide the selection of window parameters, allowing users to identify peaks corresponding to specific tissues (e.g., fat, tissue, bone) [S1, unofficial]. The technique is widely used in medical imaging (e.g., CT scans) where specific tissue windows (e.g., "Brain Window", "Tissue Window") are standard [S1, unofficial].

## Sources
[S1] PDF Direct Volume Rendering - KTH, https://www.kth.se/social/files/565e35dff27654457fb84363/08_VolumeRendering.pdf, fetched 2026-10-27 (unofficial)
[S2] Histogram spectra for multivariate time-varying volume LOD selection, https://zenodo.org/records/1275223, fetched 2026-10-26
