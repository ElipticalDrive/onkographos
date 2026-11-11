# Data Bounds Determination

Also called: Bounds Computation, Scalar Range Initialization

## Overview
Data Bounds Determination is the procedure of computing the minimum and maximum scalar values ($v_{min}, v_{max}$) for a volume dataset to enable consistent color mapping and transfer function application. It solves the problem of outlier sensitivity and cross-dataset consistency by selecting a representative scalar range from the raw data. It differs from sibling methods by offering distinct strategies for handling statistical outliers (percentiles) versus global extremes, and by supporting manual overrides for multi-file consistency.

## How it works
**Inputs:** Scalar field data (point and/or cell values); configuration specifying the method (manual, percentile, global).
**Outputs:** A scalar range $[v_{min}, v_{max}]$ used for normalization.

**Data Structures:**
*   `Histogram`: A frequency map of scalar values, bin count depends on data precision or fixed binning strategy.
*   `RangeMap`: Stores computed min/max for point and cell scalars separately.

**Steps:**
1.  **Select Method:** Determine the bounds strategy based on user configuration or default settings.
2.  **Manual Bounds:** If selected, use user-provided $v_{min}$ and $v_{max}$. This ensures consistent coloring across different file types or datasets [S1, unofficial].
3.  **Percentile Bounds:**
    *   Scan all scalar values to build a histogram [S1, unofficial].
    *   Compute cumulative distribution.
    *   Select $v_{min}$ at the lower percentile (default 1st) and $v_{max}$ at the upper percentile (default 99th) [S1, unofficial]. This excludes outliers that would compress the visible color range.
4.  **Global Bounds:**
    *   Scan all scalar values.
    *   Set $v_{min} = \min(\text{all values})$ and $v_{max} = \max(\text{all values})$ [S1, unofficial].
5.  **Union of Point and Cell Ranges (Variant):**
    *   If the dataset contains both point and cell scalars, compute ranges for each separately.
    *   If a specific scalar name is targeted, find its point range $[p_{min}, p_{max}]$ and cell range $[c_{min}, c_{max}]$.
    *   If both exist, the global range is $[\min(p_{min}, c_{min}), \max(p_{max}, c_{max})]$ [S2, unofficial].
    *   If only one type exists, use that range. If no point scalars exist, fall back to legacy mesh scalar vectors [S2, unofficial].

**Formulas:**
Color index $i$ for a value $v$ is computed as [S1, unofficial]:
$$
i = \begin{cases} 
0 & v < v_{min} \\
1 + 253 \frac{v - v_{min}}{v_{max} - v_{min}} & v_{min} \leq v \leq v_{max} \\
255 & v > v_{max}
\end{cases}
$$

**Parameters:**
*   Lower Percentile: 1 (default) [S1, unofficial].
*   Upper Percentile: 99 (default) [S1, unofficial].
*   Color Table Size: 256 entries [S1, unofficial].

**Edge Cases:**
*   **Outliers:** Global bounds may result in poor visualization if extreme outliers exist; percentile bounds mitigate this [S1, unofficial].
*   **Missing Data:** If point scalars are absent, the system must fallback to legacy scalar storage to avoid null ranges [S2, unofficial].
*   **Precision:** Histogram binning must be sufficient to resolve the percentile thresholds accurately.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def determine_bounds(data, method='percentile', lower_pct=1, upper_pct=99, manual_min=None, manual_max=None):
    """
    Determine scalar bounds for volume rendering.
    
    Args:
        data: Object containing point_scalars, cell_scalars, or legacy scalars.
        method: 'manual', 'percentile', or 'global'.
        lower_pct: Lower percentile for percentile method.
        upper_pct: Upper percentile for percentile method.
        manual_min, manual_max: User-defined bounds.
        
    Returns:
        tuple: (v_min, v_max)
    """
    if method == 'manual':
        if manual_min is None or manual_max is None:
            raise ValueError("Manual bounds require both min and max")
        return manual_min, manual_max

    # Collect all scalar values
    values = []
    if hasattr(data, 'point_scalars') and data.point_scalars:
        values.extend(data.point_scalars)
    if hasattr(data, 'cell_scalars') and data.cell_scalars:
        values.extend(data.cell_scalars)
    if not values and hasattr(data, 'legacy_scalars'):
        values.extend(data.legacy_scalars)
        
    if not values:
        return 0.0, 1.0  # [G] Default fallback if no data

    if method == 'global':
        return min(values), max(values)

    if method == 'percentile':
        # Build histogram
        hist = build_histogram(values)
        total_count = len(values)
        
        # Find lower bound
        cumulative = 0
        lower_val = values[0]
        for val, count in hist:
            cumulative += count
            if cumulative >= total_count * (lower_pct / 100.0):
                lower_val = val
                break
                
        # Find upper bound
        cumulative = 0
        upper_val = values[-1]
        for val, count in hist:
            cumulative += count
            if cumulative >= total_count * (upper_pct / 100.0):
                upper_val = val
                break
                
        return lower_val, upper_val

    raise ValueError(f"Unknown method: {method}")

def build_histogram(values, num_bins=256):
    """Simple histogram builder."""
    if not values:
        return []
    min_val = min(values)
    max_val = max(values)
    if min_val == max_val:
        return [(min_val, len(values))]
    
    bin_width = (max_val - min_val) / num_bins
    bins = [0] * num_bins
    for v in values:
        idx = int((v - min_val) / bin_width)
        if idx >= num_bins:
            idx = num_bins - 1
        bins[idx] += 1
        
    result = []
    for i, count in enumerate(bins):
        val = min_val + i * bin_width
        result.append((val, count))
    return result
```

## Subtopics

### [Local Density Bounds via Sliding Window](12750-local-density-bounds-via-sliding-window.md)
Computes minimum and maximum volume density values within a neighborhood defined by `search_radius` around each voxel. Used to establish local density bounds for volume rendering preprocessing, providing a spatially adaptive range rather than a global one [12750-local-density-bounds-via-sliding-window.md].

### Scalar Range Calculation
Computes the minimum and maximum values for all scalar fields (point and cell) and sets the global scalar range for visualization, handling unions of point and cell ranges. It iterates over mesh attributes, computes ranges using `FieldStore::computeRange`, and merges them by taking the min of mins and max of maxs if both point and cell scalars exist for a given name [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of scalar values, as it requires a single pass to compute min/max or build a histogram.
*   **Memory Complexity:** $O(N)$ for storing values or $O(B)$ for histogram bins ($B$ is bin count).
*   **Strengths:** Percentile method effectively handles outliers; manual method ensures consistency across datasets.
*   **Limitations:** Global bounds are sensitive to outliers; percentile method requires scanning all data.
*   **When to use:** Use percentile for datasets with known outliers; manual for comparative visualization; global for simple cases.

## Background
Originated in Smokeview, a tool for visualizing Fire Dynamics Simulation data. The method was documented in the NIST Special Publication 1017-2, Volume II, which details the technical reference for Smokeview Version 6 [S1, unofficial]. It addresses the need for robust scalar range initialization in fire simulation visualization where temperature and velocity data can vary widely.

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
[S2] thermeng/SciRender, https://github.com/thermeng/SciRender, fetched 2026-10-26 (unofficial)
