# Variable Plane Spacing

Also called: Bin-based Plane Distribution, Density-proportional Plane Allocation, Variable plane spacing grid construction

## Overview
Variable Plane Spacing is a coordinate grid construction method that concentrates sampling planes in regions of high data density by distributing planes proportionally across coordinate bins [S1, unofficial]. It constructs a rectangular sampling grid with non-uniform plane spacing based on input data density [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A set of scattered 3D data points with coordinates $(x, y, z)$; the desired number of planes $P$ for each axis [S1, unofficial].
*   **Output:** A rectangular sampling grid defined by non-uniform plane positions along each axis [S1, unofficial].

**Data Structures**
*   **Bin Boundaries:** An array $v$ of size $B = \lfloor P/2 \rfloor$, where $v_i$ stores the upper boundary value of bin $i$ [S1, unofficial].
*   **Bin Counts:** An integer array $C$ of size $B$, where $C[i]$ stores the count of unique coordinate values falling into bin $i$ [S1, unofficial].
*   **Plane Counts:** An integer array $N$ of size $B$, where $N[i]$ stores the number of planes to be placed within bin $i$ [S1, unofficial].
*   **Total Points:** A scalar $T$ representing the sum of all unique coordinate values across all bins [S1, unofficial].

**Steps**
1.  **Determine Extremes:** Find the minimum ($min$) and maximum ($max$) coordinate values for the axis from the input data [S1, unofficial].
2.  **Calculate Bin Parameters:** Set the number of bins $B = \lfloor P/2 \rfloor$ [S1, unofficial]. Calculate bin step $s = (max - min) / B$ [S1, unofficial]. Assign bin boundaries $v_i = min + (i+1)s$ for $i=0 \dots B-1$, ensuring $v_{B-1} = max$ [S1, unofficial].
3.  **Populate Bins:** Initialize $C[i] = 0$ [S1, unofficial]. For each unique coordinate value $c$, find the bin index $i$ such that $v_{i-1} < c \le v_i$ (with $v_{-1} = min$) [S1, unofficial]. Increment $C[i]$. Sum $C$ to obtain $T$ [S1, unofficial].
4.  **Allocate Planes:** For each bin $i$ where $C[i] > 0$, calculate planes using $N[i] = \text{round}\left(\frac{C[i]}{T} \cdot P + 0.5\right)$ [S1, unofficial].
5.  **Position Planes:** Traverse the bins. For each bin $i$ with $N[i] > 0$, position $N[i]$ planes evenly spaced from $start = v_{i-1}$ (or $min$ if $i=0$) up to $end = v_i$, excluding $end$ [S1, unofficial]. Plane $j$ ($0 \le j < N[i]$) is placed at $start + \frac{j}{N[i]}(end - start)$ [G].
6.  **Boundary Enforcement:** If a user option is enabled, force planes at exactly $min$ and $max$ regardless of bin counts to prevent grid range shrinkage [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def variable_plane_spacing(points, num_planes, force_boundaries=False):
    """
    points: list of floats (coordinates along one axis)
    num_planes: int (total planes to allocate for this axis)
    force_boundaries: bool (if True, forces planes at min and max)
    returns: list of floats (plane positions)
    """
    if not points:
        return []

    # Step 1: Determine extremes
    min_val = min(points)
    max_val = max(points)
    
    # Step 2: Calculate bin parameters
    num_bins = num_planes // 2
    if num_bins == 0:
        # Fallback to constant spacing if fewer than 2 planes requested [G]
        return [min_val + (max_val - min_val) / max(1, num_planes - 1) * i for i in range(num_planes)]
        
    bin_step = (max_val - min_val) / num_bins
    bin_boundaries = [min_val + bin_step * (i + 1) for i in range(num_bins)]
    bin_boundaries[-1] = max_val  # Ensure last bin equals coordinate maximum
    
    # Step 3: Populate bins with unique values
    unique_points = sorted(list(set(points)))
    bin_counts = [0] * num_bins
    
    for p in unique_points:
        # Find bin index i such that v[i-1] < p <= v[i]
        idx = 0
        while idx < num_bins and bin_boundaries[idx] < p:
            idx += 1
        if idx >= num_bins:
            idx = num_bins - 1  # Clamp to last bin for max value [G]
        bin_counts[idx] += 1
            
    total_points = sum(bin_counts)
    
    if total_points == 0:
        # Fallback to constant spacing if no unique points found [G]
        return [min_val + (max_val - min_val) / max(1, num_planes - 1) * i for i in range(num_planes)]

    # Step 4: Allocate planes
    plane_counts = [0] * num_bins
    for i in range(num_bins):
        if bin_counts[i] > 0:
            # np = (bp / tp) * pi + 0.5, rounded to nearest integer
            plane_counts[i] = int((bin_counts[i] / total_points) * num_planes + 0.5)
            
    # Step 5: Position planes
    planes = []
    prev_boundary = min_val
    
    for i in range(num_bins):
        count = plane_counts[i]
        if count > 0:
            current_boundary = bin_boundaries[i]
            for j in range(count):
                pos = prev_boundary + (j / count) * (current_boundary - prev_boundary)
                planes.append(pos)
        prev_boundary = current_boundary
        
    # Step 6: Boundary Enforcement
    if force_boundaries:
        if not planes or planes[0] != min_val:
            planes.insert(0, min_val)
        if not planes or planes[-1] != max_val:
            planes.append(max_val)
            
    return planes
```

**Parameters**
*   $P$: Total planes per axis. Typical values depend on desired resolution [S1, unofficial].
*   $B = P/2$: Number of bins [S1, unofficial].
*   Rounding: Add 0.5 before integer conversion to mitigate ±1 plane errors from integer truncation [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Integer Conversion Error:** The formula $np = \frac{bp}{tp} \cdot pi + 0.5$ can still yield one more or one less plane than specified due to integer conversion [S1, unofficial].
*   **Grid Range Shrinkage:** Insufficient points in the first or last bins can shrink the grid range; a user option forces planes at exact $min$ and $max$ coordinates [S1, unofficial].
*   **Extreme Size Variance:** The algorithm places too much emphasis on dense input areas, causing loss of information in less dense areas and creating cells with extreme size variances [S1, unofficial].

### Subtopics
*   **Constant Plane Spacing:** Places planes at uniform intervals along each axis. Preferred when uniform accuracy across the entire volume is required [S1, unofficial]. [Link](5069-constant-plane-spacing.md)
*   **Structured Grid Triangle List Generation:** Generates mesh topology from the constructed grid. Does not determine plane positions. [Link](8189-structured-grid-triangle-list-generation.md)

## Operating characteristics
*   **Time Complexity:** Higher than constant spacing due to maintaining linked lists of unique coordinate values and multiple traversals of the bin list [S1, unofficial].
*   **Memory Complexity:** Higher than constant spacing due to storage of bin structures and unique value lists [S1, unofficial].
*   **Measured Performance:** Requires significantly more CPU time to sample data compared to constant spacing [S1, unofficial].
*   **Strengths:** Samples data well in areas where planes are closely located to data points (high density) [S1, unofficial].
*   **Limitations:** Samples poorly in sparse areas; information from less dense areas is lost; extreme cell size variances reduce overall quality [S1, unofficial].
*   **When to Use:** When data is highly clustered and preserving detail in dense regions is prioritized over uniform coverage [S1, unofficial].
*   **When Not to Use:** When uniform accuracy across the entire volume is required, or when computational resources are limited [S1, unofficial].

## Background
The method was described in a report by the Defense Technical Information Center (DTIC) in the late 1980s/early 1990s [S1, unofficial]. It was developed as an alternative to constant plane spacing for sampling scattered data onto rectangular grids for volume visualization [S1, unofficial]. The report notes that while variable spacing has merit, the specific implementation placed too much emphasis on dense areas, suggesting that a "smarter algorithm" for spreading plane locations could yield better results [S1, unofficial].

## Sources
[S1] DTIC ADA215533: Sampling Scattered Data Onto Rectangular Grids for Volume Visualization, https://archive.org/details/DTIC_ADA215533, fetched 2026-10-26 (unofficial)
