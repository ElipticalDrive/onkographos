# Data Bounds Determination

Also called: Bounds Computation, Scalar Range Initialization

## Overview
Data Bounds Determination is a procedure to obtain minimum and maximum data bounds for scaling colors in volume rendering [S1, unofficial]. It solves the problem of establishing the correct data range for visualization by selecting between manual, percentile, or global bounds [S1, unofficial]. It differs from [Scalar Range Calculation](11487-scalar-range-calculation.md) by focusing on the specific selection strategy (including outlier handling via percentiles) rather than just computing the raw global extrema of point and cell data.

## How it works
**Inputs and outputs:**
- Input: The scalar dataset values.
- Output: A minimum bound ($v_{min}$) and a maximum bound ($v_{max}$) used for color mapping.

**Data structures:**
- A histogram of the data values is computed to support percentile determination [S1, unofficial].

**Steps:**
The method selects one of three strategies [S1, unofficial]:

1. **Manual Bounds:** The user explicitly sets $v_{min}$ and $v_{max}$. This is used to ensure consistent coloring when multiple file types (e.g., slices and boundary files) are displayed simultaneously [S1, unofficial].
2. **Percentile Bounds:** This method handles data outliers. The algorithm scans the data to compute a histogram [S1, unofficial]. It then selects $v_{min}$ and $v_{max}$ at specified percentile levels. The default percentiles are the 1st and 99th [S1, unofficial].
3. **Global Bounds:** The algorithm selects the absolute minimum and maximum values for the entire dataset [S1, unofficial].

**Formulas:**
While the determination step selects $v_{min}$ and $v_{max}$, the subsequent color index calculation uses these bounds [S1, unofficial]:

$$
\text{color index} = 
\begin{cases} 
0 & v < v_{min} \\
1 + 253(v - v_{min})/(v_{max} - v_{min}) & v_{min} \leq v \leq v_{max} \\
255 & v > v_{max}
\end{cases}
$$

**Parameters:**
- Default percentiles: 1st and 99th [S1, unofficial].
- Color table size: 256 entries [S1, unofficial].

**Edge cases and pitfalls:**
- Outliers: Using global bounds (Method 3) can compress the visible range if outliers are present; percentile bounds (Method 2) mitigate this [S1, unofficial].
- Consistency: Manual bounds (Method 1) are required when comparing different data types that must share the same color scale [S1, unofficial].

## Subtopics

### [Scalar Range Calculation](11487-scalar-range-calculation.md)
Computes the global minimum and maximum values for all scalar fields (both point and cell data) to establish the visualization scalar range. It solves the problem of determining the correct data bounds for color and opacity mapping by handling the union of potentially disjoint point and cell scalar ranges. It differs from [Local Density Bounds via Sliding Window](12750-local-density-bounds-via-sliding-window.md) by computing global extrema rather than local density statistics.

### [Local Density Bounds via Sliding Window](12750-local-density-bounds-via-sliding-window.md)
This method computes the minimum and maximum volume density values within a neighborhood defined by a `search_radius` around each voxel, establishing local density bounds for volume rendering preprocessing. It differs from global scalar range calculation by providing spatially varying bounds that adapt to local data density, and from naive box filtering by achieving linear time complexity $O(N)$ using monotonic queues instead of $O(N \cdot R^3)$.

## Operating characteristics
Not given in the sources.

## Background
This procedure is described in the context of Smokeview, a tool for visualizing Fire Dynamics Simulation data [S1, unofficial].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
