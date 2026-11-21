# Bjøntegaard Delta-Rate (BD-Rate)

Also called: BD-R, BD-Rate, Piecewise cubic Bjøntegaard Delta-Rate (BD-Rate) metric

## Overview
BD-Rate is a metric for fair comparison of overall rate-distortion (RD) performance that calculates the average bit-rate saving at a given quality level between two methods [S1]. It sets itself apart from point-wise comparisons by integrating the area between two continuous RD curves, providing a single percentage value representing efficiency gains [S1]. It builds on the concept of BD-PSNR but inverts the perspective to measure bitrate reduction [7176-bj-ntegaard-delta-psnr-bd-psnr-and-bd-br.md].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Two sets of RD points for Method A and Method B. Each set consists of pairs $(R_i, D_i)$, where $R$ is the bitrate (e.g., bits per pixel) and $D$ is the distortion metric (e.g., PSNR in dB).
*   **Output:** A single percentage value representing the average bitrate saving. A negative value indicates Method A uses less bitrate than Method B for the same quality.

**Data Structures:**
*   Two lists of tuples: `points_A = [(R1, D1), (R2, D2), ...]` and `points_B = [(R1, D1), (R2, D2), ...]`.
*   Points must be sorted by Distortion ($D$) in ascending order.

**Steps:**
1.  **Log-Transform:** Convert bitrate values to natural logarithms. Let $x = \ln(R)$ and $y = D$. This linearizes the typically exponential relationship between bitrate and quality [S1].
2.  **Piecewise Cubic Interpolation:** Fit a continuous cubic spline curve through the $(x, y)$ points for each method. The spline ensures $C^2$ continuity (continuous first and second derivatives) [S1].
3.  **Integration:** Calculate the area between the two interpolated curves over the range of distortion values covered by both methods. The integral is computed as:
    $$ \Delta R = \int_{y_{min}}^{y_{max}} (x_A(y) - x_B(y)) \, dy $$
    where $x_A(y)$ and $x_B(y)$ are the interpolated log-bitrate functions for Method A and Method B, respectively, and $[y_{min}, y_{max}]$ is the intersection of their distortion ranges [S1].
4.  **Normalization:** Divide the integrated area by the range of distortion $(y_{max} - y_{min})$ to get the average difference in log-bitrate [S1].
5.  **Exponentiation:** Convert the average log-bitrate difference back to a linear scale to obtain the BD-Rate percentage:
    $$ BD\text{-}Rate = 100 \times (e^{\Delta R / \Delta y} - 1) $$
    where $\Delta y = y_{max} - y_{min}$ [S1].

**Parameters:**
*   **Interpolation Method:** Piecewise cubic spline. Standard cubic splines may oscillate; piecewise cubic ensures stability across the RD curve [S1].
*   **Distortion Metric:** Typically PSNR (dB) [S1].
*   **Rate Metric:** Bits per pixel (bpp) or model size in MB [S1].

**Edge Cases and Pitfalls:**
*   **Non-overlapping Ranges:** If the two methods do not cover the same range of distortion, the integration limits are restricted to the intersection of their $D$ ranges. Extrapolation is not performed [S1].
*   **Precision:** Log-transforming small bitrate values can lead to numerical instability if $R \le 0$. Ensure $R > 0$ [G].
*   **Monotonicity:** RD curves should be monotonic. If a method produces non-monotonic RD points (e.g., higher bitrate for lower quality), the interpolation may produce artifacts. Pre-processing to remove non-Pareto-optimal points is recommended [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.interpolate import CubicSpline

def calculate_bjontegaard_delta_rate(points_a, points_b):
    """
    Calculate BD-Rate between two rate-distortion curves.
    
    Args:
        points_a: List of tuples (bitrate, psnr) for method A.
        points_b: List of tuples (bitrate, psnr) for method B.
        
    Returns:
        float: BD-Rate percentage.
    """
    # 1. Sort by PSNR (distortion)
    points_a = sorted(points_a, key=lambda p: p[1])
    points_b = sorted(points_b, key=lambda p: p[1])
    
    # 2. Extract arrays
    r_a, d_a = zip(*points_a)
    r_b, d_b = zip(*points_b)
    
    # 3. Log-transform bitrate
    x_a = np.log(r_a)
    x_b = np.log(r_b)
    
    # 4. Determine integration range (intersection of PSNR ranges)
    d_min = max(min(d_a), min(d_b))
    d_max = min(max(d_a), max(d_b))
    
    if d_min >= d_max:
        return 0.0 # No overlap
        
    # 5. Create cubic splines
    # Note: CubicSpline requires at least 4 points for full cubic, 
    # but scipy handles fewer with 'not-a-knot' or similar boundary conditions.
    # For robustness, ensure sufficient points.
    cs_a = CubicSpline(d_a, x_a)
    cs_b = CubicSpline(d_b, x_b)
    
    # 6. Integrate the difference between curves
    # We integrate from d_min to d_max
    # Using numerical integration (trapezoidal rule) for simplicity and robustness
    num_points = 1000
    d_int = np.linspace(d_min, d_max, num_points)
    x_diff = cs_a(d_int) - cs_b(d_int)
    
    # Area under the curve of the difference
    area = np.trapz(x_diff, d_int)
    
    # 7. Calculate average log-bitrate difference
    delta_y = d_max - d_min
    avg_log_diff = area / delta_y
    
    # 8. Convert to percentage
    bd_rate = 100.0 * (np.exp(avg_log_diff) - 1.0)
    
    return bd_rate
```

## Subtopics

### Rate-Distortion Performance Evaluation
This is the primary application context for BD-Rate. It is used to compare compression algorithms (e.g., SPC-NeRF vs. VQRF) by evaluating their overall efficiency across multiple quality levels [S1]. It is chosen when a single scalar metric is needed to summarize the trade-off between model size/bitrate and rendering quality (PSNR) [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** $O(N \log N)$ for sorting, where $N$ is the number of RD points. Spline fitting and integration are $O(N)$ or $O(N \log N)$ depending on the implementation.
*   **Memory:** $O(N)$ to store the RD points and spline coefficients.

**Measured Performance:**
*   In comparisons of SPC-NeRF against VQRF on the Synthetic-NeRF dataset, BD-Rate showed a 28% bit saving for the base configuration and 32% for the HR configuration [S1].
*   BD-Rate is sensitive to the range of distortion covered. If one method significantly outperforms another only at very high or very low bitrates, the BD-Rate may not reflect the practical utility if the operating range is limited [S1].

**Strengths and Limitations:**
*   **Strengths:** Provides a single, comparable metric for overall RD performance. Accounts for the entire curve, not just a single operating point.
*   **Limitations:** Assumes a smooth, continuous RD curve. Discrete, sparse RD points can lead to inaccurate interpolation. It does not indicate *where* the gains occur (e.g., low bitrate vs. high bitrate) [S1].

**When to Use:**
*   Use when comparing two compression methods across a wide range of bitrates and quality levels [S1].
*   Use when a single percentage metric is required for ablation studies or final reporting [S1].

**When Not to Use:**
*   Do not use if the RD curves do not overlap significantly in distortion range.
*   Do not use if the RD points are too sparse to form a reliable cubic spline [S1].

## Background
BD-Rate was introduced by Bjøntegaard in the context of video compression standards [S1]. It has been adapted for evaluating neural radiance field compression methods, such as SPC-NeRF, to provide a fair comparison against prior works like VQRF [S1]. The metric relies on piecewise cubic interpolation to estimate the continuous RD curve from discrete experimental points [S1].

## Sources
[S1] SPC-NeRF: Spatial Predictive Compression for Voxel-Based Radiance Field, https://doi.org/10.1145/3795528, fetched 2026-10-26
[S2] Distilled Low Rank Neural Radiance Field with Quantization for Light Field Compression, https://core.ac.uk/works/391353499, fetched 2026-10-30 (unofficial)
