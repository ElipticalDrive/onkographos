# Alpha-histogram Peak Analysis Scheme

Also called: Automatic Peak Detection in Alpha-histogram, Peak Detection Scheme, Peak analysis scheme

## Overview
The Alpha-histogram Peak Analysis Scheme is an automatic method for detecting significant material peaks in an $\alpha$-histogram to facilitate transfer function design [S1]. It addresses the challenge of noise in $\alpha$-histograms by applying selective and global smoothing, followed by an area-based importance assessment to filter and simplify the remaining peaks [S1].

## How it works
**Inputs:** An $\alpha$-histogram represented as an array of bin heights [S1].
**Outputs:** A list of up to $np_2$ significant peaks, where each peak is defined by its apex position, width, and area [S1].

**Data Structures:**
*   **Histogram:** An array $H$ of size $N$, where $H[i]$ is the count for intensity bin $i$.
*   **Peak:** A tuple $(x_0, x_-, x_+, A)$, where $x_0$ is the apex index, $x_-$ and $x_+$ are the 10th and 90th percentile width boundaries, and $A$ is the calculated area [S1].

**Parameters:**
*   $np_1$: Maximum number of peaks allowed after global smoothing. Typical value: 20 [S1].
*   $np_2$: Predefined number of peaks to retain after simplification. Typical values: 1, 2, 3, 4 [S1].

**Algorithm Steps:**

1.  **Selective Smoothing:**
    Identify "minimal peaks" or "creases". A minimal peak is an apex surrounded by two valleys (valley-apex-valley). A crease is an apex surrounded by two valleys in the pattern apex-valley-apex [S1]. Specifically, a point $x_i$ is a minimal peak or crease if it is the middle of three directly adjacent valley-apex-valley or apex-valley-apex points [S1]. Apply a binomial smoothing kernel to these specific points. Repeat this process iteratively until no minimal peaks or creases remain in the histogram [S1].

2.  **Global Smoothing:**
    Apply the same binomial smoothing kernel to the entire histogram array [S1]. Repeat the smoothing process. Stop when the number of remaining peaks in the histogram equals $np_1$ [S1].

3.  **Peak Listing and Simplification:**
    Identify all remaining peaks in the smoothed histogram. Calculate the area for each peak (see Formula below). While the number of peaks exceeds $np_2$: find the peak with the least area, remove it from the list, and extend a neighboring peak to cover the removed peak's range if doing so increases the neighbor's area [S1]. If both left and right neighbors would gain from an extension, select the neighbor with the largest peak height [S1].

**Formulas:**

*   **Smoothing Kernel:**
    The smoothing operator is a binomial kernel applied to the histogram bins [S1]:
    $$ K = [0.25, 0.5, 0.25] $$
    For a point $x_i$, the smoothed value $H'[i]$ is:
    $$ H'[i] = 0.25 \cdot H[i-1] + 0.5 \cdot H[i] + 0.25 \cdot H[i+1] $$
    (Boundary conditions for $i=0$ and $i=N-1$ are not specified in the sources [G]. Standard practice is to clamp or reflect, but the source does not define this.)

*   **Peak Definition:**
    An apex is a point $x_0$ such that $H[x_0] > H[x_0-1]$ and $H[x_0] > H[x_0+1]$. A valley is the opposite [S1].

*   **Base Line Construction:**
    For a peak with apex $x_0$, the base line starts at the highest valley with a downward slope relative to the peak. It extends as a straight line touching exactly one more point on the histogram curve within the peak range [S1].

*   **Peak Area:**
    The area $A$ is the sum of the height differences between the base line and the histogram curve between the two touching points (the start valley and the second touching point) [S1].
    $$ A = \sum_{j=start}^{end} (H_{base}(j) - H(j)) $$
    Where $H_{base}(j)$ is the value of the base line at bin $j$.

*   **Peak Width (for evaluation, not core scheme):**
    $x_-$ and $x_+$ are defined as the 10th and 90th percentiles of the peak histogram area [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def alpha_histogram_peak_analysis(H, np1, np2):
    """
    Implements the Alpha-histogram Peak Analysis Scheme.
    
    Args:
        H: List of float/int, the bin counts.
        np1: int, max peaks after global smoothing.
        np2: int, final number of peaks to retain.
        
    Returns:
        List of Peak objects.
    """
    N = len(H)
    
    # Helper predicates
    def is_apex(i): return H[i] > H[i-1] and H[i] > H[i+1]
    def is_valley(i): return H[i] < H[i-1] and H[i] < H[i+1]
    
    # Step 1: Selective Smoothing
    while True:
        changed = False
        for i in range(1, N - 1):
            # Check for valley-apex-valley or apex-valley-apex centered at i
            if (is_valley(i-1) and is_apex(i) and is_valley(i+1)) or \
               (is_apex(i-1) and is_valley(i) and is_apex(i+1)):
                H[i] = 0.25 * H[i-1] + 0.5 * H[i] + 0.25 * H[i+1]
                changed = True
        if not changed:
            break
            
    # Step 2: Global Smoothing
    def count_peaks(arr):
        return sum(1 for i in range(1, len(arr)-1) if arr[i] > arr[i-1] and arr[i] > arr[i+1])
        
    while count_peaks(H) > np1:
        H = [H[0]] + [0.25*H[i-1] + 0.5*H[i] + 0.25*H[i+1] for i in range(1, N-1)] + [H[N-1]]
        
    # Step 3 & 4: Peak Listing & Simplification
    class Peak:
        def __init__(self, apex_idx, area, height):
            self.apex_idx = apex_idx
            self.area = area
            self.height = height
            
    def compute_peak_area(start, end, apex):
        # Base line: start at highest valley with downward slope
        left = apex
        while left > start and H[left] < H[left-1]: left -= 1
        # Extend line touching exactly one more point
        right = apex
        while right < end and H[right] < H[right+1]: right += 1
        # Calculate area between base line and curve
        area = 0.0
        for j in range(left, right + 1):
            t = (j - left) / max(1, right - left)
            base_val = H[left] + t * (H[right] - H[left])
            area += max(0.0, base_val - H[j])
        return area

    peaks = []
    for i in range(1, N-1):
        if is_apex(i):
            # Find peak boundaries
            l, r = i, i
            while l > 0 and H[l] >= H[l-1]: l -= 1
            while r < N-1 and H[r] >= H[r+1]: r += 1
            area = compute_peak_area(l, r, i)
            peaks.append(Peak(i, area, H[i]))
            
    while len(peaks) > np2:
        min_idx = min(range(len(peaks)), key=lambda k: peaks[k].area)
        removed = peaks.pop(min_idx)
        
        candidates = []
        if min_idx > 0: candidates.append(peaks[min_idx-1])
        if min_idx < len(peaks): candidates.append(peaks[min_idx])
        
        best = None
        max_gain = -1.0
        for cand in candidates:
            # Estimate area gain by extending range to cover removed peak
            new_l = min(cand.apex_idx, removed.apex_idx)
            new_r = max(cand.apex_idx, removed.apex_idx)
            new_area = compute_peak_area(new_l, new_r, cand.apex_idx)
            gain = new_area - cand.area
            if gain > max_gain:
                max_gain = gain
                best = cand
            elif gain == max_gain and best is not None and cand.height > best.height:
                best = cand
                
        if best and max_gain > 0:
            best.area = best.area + max_gain
            
    return peaks
```

## Subtopics

### [Partial Range Histogram (PRH)](12656-partial-range-histogram-prh.md)
The Partial Range Histogram (PRH) method creates separate histograms for individual tissues by analyzing local voxel neighborhoods, contrasting with the $\alpha$-histogram which enhances tissues at the global histogram level. It is specifically designed to detect highly overlapping peaks and expose minor tissue peaks that are obscured in the global histogram.

### [TF adaptation peak selection](12678-tf-adaptation-peak-selection.md)
This method selects a single peak from a set of candidates detected in an original or $\alpha$-histogram by prioritizing those within a predefined valid range and maximizing a confidence score $c_p$. It differs from its parent, [Alpha-histogram Peak Analysis Scheme](12652-alpha-histogram-peak-analysis-scheme.md), by applying this validity and confidence filter, and from siblings like [Partial Range Histogram (PRH)](12656-partial-range-histogram-prh.md) by relying on $c_p$ rather than partial range constraints alone.

### [Exhaustive Multiple-Peak Search](14136-exhaustive-multiple-peak-search.md)
An automatic algorithm to detect all tissue peaks in a global histogram by iteratively finding the highest peak, fitting a Gaussian, creating a PRH for that peak's range, removing the PRH's contribution from the global histogram, and repeating until the histogram is empty.

## Operating characteristics
**Time Complexity:** Not given in the sources.
**Memory Complexity:** Not given in the sources.
**Performance:** The scheme is designed to handle noisy $\alpha$-histograms effectively [S1]. The selective smoothing step preserves subtle peaks that might be removed by global smoothing alone [S1].
**Strengths:** Automatic detection; handles noise; area-based importance is intuitive for material significance [S1].
**Limitations:** Requires tuning of $np_1$ and $np_2$ [S1]. Intensity value 0 is often discarded as it is orders of magnitude more frequent [S1].

## Background
The Alpha-histogram Peak Analysis Scheme was introduced in the paper "The alpha-histogram: Using Spatial Coherence to Enhance Histograms and Transfer Function Design" [S1]. It is part of a pipeline for automatic transfer function generation, building on the $\alpha$-histogram enhancement technique. The scheme was evaluated on 20 MR AAA angiographies [S1].

## Sources
[S1] The alpha -histogram: Using Spatial Coherence to Enhance Histograms and Transfer Function Design, https://doi.org/10.2312/vissym/eurovis06/227-234, fetched 2026-10-26
