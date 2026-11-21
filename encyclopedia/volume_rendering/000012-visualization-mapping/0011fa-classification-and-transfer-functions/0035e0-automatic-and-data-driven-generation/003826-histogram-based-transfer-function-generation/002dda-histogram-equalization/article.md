# Histogram Equalization

Also called: Automatic Range Mapping, CDF-based Normalization

## Overview
Histogram Equalization is an automatic technique that redistributes scalar values by spreading out clustered regions of the data distribution to achieve a uniform output distribution [S2, unofficial]. It solves the problem of visualizing high dynamic range (HDR) data where features are hidden in dense spikes of the scalar histogram, by mapping input values to a normalized range where every intensity level is equally probable [S2, unofficial]. Unlike its sibling method, [Range Mapping](5650-range-mapping.md), which uses piecewise linear functions to provide user control and maintain continuity, Histogram Equalization is fully automatic and tends to break the continuity of the scalar range [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A normalized scalar value $x$ from JND-corrected volume data in the range $[0, 1]$ [S1][S2, unofficial].
*   Output: A uniformly distributed normalized scalar $y$ in the range $[0, 1]$ [S2, unofficial].

**Data Structures:**
*   **Histogram:** A discrete array of size $N$ storing the count of voxels falling into each bin [G].
*   **CDF Array:** An array of size $N$ storing the cumulative sums of the histogram bins [G].

**Steps:**
1.  Construct the normalized cumulative histogram of the JND-corrected volume data [S1].
2.  Compute the Probability Density Function (PDF) by counting the occurrences of each scalar value or binned value in the dataset [G].
3.  Compute the Cumulative Distribution Function (CDF) as a sum of PDFs over normalized scalar inputs: $CDF(x_i) = \sum_{x_j < x_i} PDF(x_j)$ [S2, unofficial].
4.  Map the normalized scalar input $x$ to the uniformly distributed output $y$: $y = CDF(x)$ [S2, unofficial].

**Parameters:**
*   The method is applied to six order-independent operators: summation and maximum applied to original data values, gradient magnitude, and the product of data value and gradient magnitude [S1].
*   Number of bins $N$: Not given in the sources [G].

**Edge Cases and Pitfalls:**
*   The theoretical motivation does not strictly apply to volume rendering because the perceptually linear mapping is followed by a non-linear mapping via exponential fall-off from the volume rendering integral, though it still provides a good starting point [S1].
*   For the maximum operator, the histogram of volume densities is not a good predictor for pixel values because the operator is view-dependent; thus, image-space equalization is used instead of data-space [S1].
*   The method gives the user very little control over the redistribution process and tends to break the continuities of the scalar range [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def histogram_equalization(scalar_data, num_bins):
    """
    Performs histogram equalization on JND-corrected volume data.
    
    Args:
        scalar_data: 1D array of normalized scalar values in [0, 1].
        num_bins: Integer, number of bins for the histogram.
        
    Returns:
        equalized_data: 1D array of equalized values in [0, 1].
        cdf: 1D array of size num_bins, the cumulative distribution function.
    """
    # Step 1: Compute Histogram (PDF)
    histogram = [0] * num_bins
    
    for value in scalar_data:
        bin_index = int(value * num_bins)
        if bin_index >= num_bins:
            bin_index = num_bins - 1
        histogram[bin_index] += 1
        
    total_voxels = len(scalar_data)
    pdf = [count / total_voxels for count in histogram]
    
    # Step 2: Compute CDF
    # Source formula: CDF(x_i) = sum(PDF(x_j) for x_j < x_i) [S2, unofficial]
    cdf = [0.0] * num_bins
    for i in range(num_bins):
        cdf[i] = sum(pdf[:i])
        
    # Normalize CDF to [0, 1] to cover the full range [G]
    max_cdf = cdf[-1] if cdf[-1] > 0 else 1.0
    cdf = [val / max_cdf for val in cdf]
        
    # Step 3: Map Input to Output
    equalized_data = [0.0] * len(scalar_data)
    for i, value in enumerate(scalar_data):
        bin_index = int(value * num_bins)
        if bin_index >= num_bins:
            bin_index = num_bins - 1
        equalized_data[i] = cdf[bin_index]
        
    return equalized_data, cdf
```

## Subtopics

### [Range Mapping](5650-range-mapping.md)
Range mapping is a technique that redistributes scalar values non-linearly to spread regions of interest evenly across a fixed-size lookup table, addressing the resolution limitations of hardware lookup tables for high dynamic range (HDR) data. Unlike its sibling method, [Histogram Equalization](5642-histogram-equalization.md), which automatically redistributes values and breaks continuity, range mapping uses piecewise linear functions to provide user control while maintaining the continuity and order of the scalar range. It exploits the non-linear shape of the histogram to magnify specific data clusters, allowing users to focus on narrow clusters of scalar values without the resolution artifacts inherent in fixed-size textures.

### [Histogram Population (Normal & Equalized)](15394-histogram-population-normal-equalized.md)
Histogram Population generates the underlying intensity distribution data structures required for transfer function generation in volume rendering [G]. It solves the problem of converting raw scalar voxel values into a normalized, visualizable format (line segments) that represents either the raw frequency of intensities (Normal) or their cumulative distribution (Equalized) [G]. It differs from its parent [Histogram Equalization](5642-histogram-equalization.md) by focusing on the construction of the histogram geometry itself rather than just the remapping logic, and from [Range Mapping](5650-range-mapping.md) by providing the binned density data necessary for automatic transfer function design [G].

### Histogram Stretching with Percentile Clipping
Histogram Stretching with Percentile Clipping is a post-processing step to create a balanced RGB image by suppressing high intensity outliers caused by constructive interference phenomena inherent to diffraction [S3]. Percentile clipping is applied to suppress high intensity outliers [S3]. The maximum intensity $I_{max}$ per color channel is determined by the maximum intensity value after clipping the higher values belonging to the high 0.15 percentile [S3]. Histogram stretching modifies the intensity range using the formula: $I(x, y) = \frac{I(x, y) - I_{min}}{I_{max} - I_{min}}$, where $I_{min}$ and $I_{max}$ are the minimum and maximum values of the intensity respectively [S3].

## Operating characteristics
*   **Time Complexity:** $O(N)$ for computing the histogram and CDF, where $N$ is the number of voxels [G]. The mapping step is also $O(N)$ or $O(1)$ per voxel if using a lookup table [G].
*   **Memory Complexity:** $O(N)$ for storing the input and output data [G]. $O(B)$ for the histogram and CDF, where $B$ is the number of bins [G].
*   **Strengths:** Simple, fast, and automatic [G]. Effective at enhancing features hidden in clustered data regions [S2, unofficial].
*   **Limitations:** Breaks continuity of the scalar range, leading to potential visual artifacts [S2, unofficial]. Provides no user control over the specific redistribution [S2, unofficial].
*   **When to use:** When automatic enhancement of HDR data is needed and user control is not required [G].
*   **When not to use:** When maintaining the continuity of the scalar range is critical, or when user control over the transfer function is desired (use [Range Mapping](5650-range-mapping.md) instead) [G].

## Background
Histogram Equalization was introduced as an automatic approach to redistribute scalar values by spreading out clustered regions, used as a basis for range mapping but providing less user control [S2, unofficial]. It is commonly used in volume rendering to handle high dynamic range (HDR) data, where the non-uniform distribution of scalar values makes feature visualization difficult [S2, unofficial]. The method is described in the context of earthquake simulation and other scientific simulations [S2, unofficial].

## Sources
[S1] Volume Rendering for High Dynamic Range Displays, https://vccimaging.org/Publications/Ghosh2005VRH/Ghosh2005VRH.pdf, fetched 2026-10-27
[S2] PDF Interactive Transfer Function Specification for Direct Volume Rendering ..., https://www.sci.utah.edu/~stevec/papers/transfer_functions.pdf, fetched 2026-10-28 (unofficial)
[S3] Three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays, https://doi.org/10.1117/12.2237581, fetched 2026-11-04
