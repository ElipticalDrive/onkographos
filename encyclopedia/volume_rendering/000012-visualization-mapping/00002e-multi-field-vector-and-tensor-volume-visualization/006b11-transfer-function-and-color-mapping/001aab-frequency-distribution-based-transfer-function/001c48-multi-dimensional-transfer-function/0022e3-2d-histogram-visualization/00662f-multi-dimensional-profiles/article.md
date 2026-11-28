# Multi-dimensional Profiles

Also called: Weighted Histograms

## Overview
Multi-dimensional Profiles compute weighted averages or distributions of a dependent variable with respect to one, two, or three independent variables, effectively creating a binned profile in phase space [S1]. Unlike [2D Histogram Visualization](2863-2d-histogram-visualization.md), which visualizes the count of voxels per bin, Multi-dimensional Profiles allow the bin values to represent the weighted sum or average of an additional quantity (e.g., mass, temperature), enabling the analysis of physical properties within specific regions of the independent variable space [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A data container (set of points/voxels), up to three independent variables (axes $x, y, z$), one dependent variable to be profiled, and bin boundaries for each axis.
*   **Outputs:** For each bin $j$ along the defined axes, the weighted average $V_j$ or the unweighted sum of the dependent variable for points falling into that bin.

**Data Structures:**
The implementation uses classes such as `BinnedProfile1D`, `BinnedProfile2D`, or `BinnedProfile3D` [S1]. Each class stores:
*   Bin boundaries for each axis: $x_0, \dots, x_{n-1}$ (where $n$ is the number of bins) [S1].
*   Accumulators for the weighted sum and the weight sum for each bin [S1].

**Steps:**
1.  **Define Axes:** Define up to three independent axes of comparison ($x, y, z$). These are distinct from spatial coordinates and represent physical quantities (e.g., density, temperature) [S1]. Discretize each axis into bins defined by boundaries $x_0, \dots, x_{n-1}$ [S1].
2.  **Bin Assignment:** For each point $i$ in the dataset with value $v_i$ along a specific axis, determine the bin index $j$ such that:
    $$ x_j \le v_i < x_{j+1} $$
    [S1]
3.  **Accumulation:** Calculate the value for each bin $j$ using the points assigned to it. Let $N$ be the number of points in bin $j$, $v_i$ be the dependent variable value for point $i$, and $w_i$ be the weight for point $i$. The weighted average $V_j$ is:
    $$ V_j = \frac{\sum_{i=1}^{N} v_i w_i}{\sum_{i=1}^{N} w_i} $$
    [S1]
    *   **Unweighted Profile (Histogram):** To compute a non-averaged distribution (count or total mass), set all weights $w_i = 1.0$ in the numerator and omit the denominator calculation (effectively summing $v_i$ directly) [S1].
4.  **Extension:** This process is trivially extended to multiple dimensions by assigning points to multi-dimensional bins based on their indices along all defined axes [S1].

**Parameters:**
*   **Weights ($w_i$):** For weighted averages, weights typically represent cell mass or volume [S1]. For unweighted distributions, weights are implicitly 1.0 [S1].
*   **Bin Boundaries:** Defined by the user for each independent variable [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_multi_dimensional_profile(points, independent_vars, dependent_var, bin_edges, weighted=True):
    """
    Compute multi-dimensional profiles.
    
    Args:
        points: List of point indices or objects.
        independent_vars: List of arrays for each independent axis (x, y, z).
        dependent_var: Array of values to profile.
        bin_edges: List of arrays defining bin boundaries for each axis.
        weighted: Boolean. If False, computes unweighted sum (histogram).
    
    Returns:
        Dictionary mapping bin indices to computed values.
    """
    num_axes = len(independent_vars)
    num_bins_per_axis = [len(edges) - 1 for edges in bin_edges]
    
    # Initialize accumulators
    # Structure: dict mapping tuple of bin indices to (sum_val, sum_weight)
    bin_accumulators = {}
    
    for i, point in enumerate(points):
        # 1. Determine bin indices for each axis
        bin_indices = []
        for axis_idx, axis_values in enumerate(independent_vars):
            val = axis_values[i]
            edges = bin_edges[axis_idx]
            # Find j such that edges[j] <= val < edges[j+1]
            # In practice, use searchsorted or similar logic
            j = find_bin_index(val, edges)
            if j is not None:
                bin_indices.append(j)
            else:
                # Point falls outside defined range, skip or handle edge case
                continue
        
        if len(bin_indices) != num_axes:
            continue
            
        # 2. Get dependent variable and weight
        v_i = dependent_var[i]
        w_i = 1.0 if not weighted else get_weight_for_point(point) # Weight logic depends on data source [S1]
        
        # 3. Accumulate
        key = tuple(bin_indices)
        if key not in bin_accumulators:
            bin_accumulators[key] = [0.0, 0.0] # [sum_vw, sum_w]
            
        bin_accumulators[key][0] += v_i * w_i
        if weighted:
            bin_accumulators[key][1] += w_i
            
    # 4. Compute final values
    results = {}
    for key, (sum_vw, sum_w) in bin_accumulators.items():
        if weighted:
            if sum_w == 0:
                results[key] = 0.0
            else:
                results[key] = sum_vw / sum_w
        else:
            # Unweighted: sum_vw is just sum of v_i (since w_i=1)
            results[key] = sum_vw
            
    return results

def find_bin_index(val, edges):
    """Find bin index j such that edges[j] <= val < edges[j+1]"""
    # Implementation detail: iterate or use binary search
    for j in range(len(edges) - 1):
        if edges[j] <= val < edges[j+1]:
            return j
    return None
```

## Subtopics

### Weighted Average Profile
A method that calculates the distribution of a quantity as a function of one or more axes by binning data points and computing weighted averages within each bin [S1]. It differs from simple averaging by using weights (e.g., cell mass or volume) to account for varying cell sizes, allowing for the examination of mass distribution in phase space rather than just point count [S1]. Choose this when the physical quantity of interest is distributed unevenly across cells with different volumes or masses [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of points $N$ and the number of bins, as each point is assigned to a bin and accumulators are updated [S1].
*   **Memory Complexity:** Proportional to the number of bins in the multi-dimensional space [S1].
*   **Strengths:** Allows for the analysis of physical properties (like mass distribution) in phase space defined by non-spatial variables (e.g., density vs. temperature) [S1].
*   **Limitations:** Computationally expensive if the number of bins is large or if the data container is massive, as it requires iterating over all points [S1].
*   **Use Cases:** Astrophysical simulations (e.g., galaxy mergers) to analyze equation of state, molecular hydrogen fractions, or velocity distributions [S1].

## Background
The concept originates from the analysis of astrophysical simulation data, where multi-dimensional histograms are constructed to study phenomena such as the mass of collapse and effective equation of state [S1]. The term "profile" is used in the `yt` toolkit to describe these weighted averages or distributions [S1]. The method was formalized in the context of the `yt` analysis toolkit, which provides `BinnedProfile` classes for 1D, 2D, and 3D profiles [S1].

## Sources
[S1] YT: A Multi-Code Analysis Toolkit for Astrophysical Simulation Data, https://digital.library.unt.edu/ark:/67531/metadc834104/?q=fluid simulation volume rendering, fetched 2026-10-28
