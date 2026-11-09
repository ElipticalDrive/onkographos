# Histogram Equalization

Also called: Automatic Range Mapping, CDF-based Normalization

## Overview
Histogram Equalization is an automatic technique that redistributes scalar values in volume rendering data to spread out clustered regions, thereby enhancing features hidden within high-density areas of the data distribution [S1, unofficial]. It differs from its sibling method, [Range Mapping](5650-range-mapping.md), by being fully automatic and lacking user control, which often results in breaking the continuity of the scalar range [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A normalized scalar value $x$ derived from the volume data, typically represented via a discrete histogram [S1, unofficial].
*   **Output:** A new normalized scalar value $y$ that is uniformly distributed across the output range [S1, unofficial].

**Data Structures:**
*   **Histogram:** A discrete representation of the Probability Density Function (PDF) across $N$ bins, where $PDF(x_i)$ is the density for bin $i$ [S1, unofficial].
*   **Lookup Table (LUT):** A precomputed array mapping input indices to output values, used for speed [S1, unofficial].

**Steps:**
1.  **Compute PDF:** Calculate the probability density for each normalized scalar input $x_i$ [S1, unofficial]. In a discrete implementation, this is the count of voxels falling into bin $i$ divided by the total number of voxels [S1, unofficial].
2.  **Compute CDF:** Calculate the Cumulative Distribution Function (CDF) by summing the PDFs [S1, unofficial]. For a normalized scalar input $x_i$, the CDF is defined as:
    $$CDF(x_i) = \sum_{x_j < x_i} PDF(x_j)$$
    The source formula uses a strict inequality ($x_j < x_i$) for the summation limit [S1, unofficial].
3.  **Map:** Perform the mapping on the normalized scalar input value $x$ to yield the output $y$:
    $$y = CDF(x)$$
    The source notes that a simple mapping is performed to yield the uniformly distributed normalized output [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def histogram_equalization(volume_data, num_bins):
    # Step 1: Compute Histogram (PDF)
    min_val = min(volume_data)
    max_val = max(volume_data)
    normalized_data = [(v - min_val) / (max_val - min_val) for v in volume_data]
    
    histogram = [0] * num_bins
    for val in normalized_data:
        bin_index = int(val * (num_bins - 1))
        if bin_index >= num_bins:
            bin_index = num_bins - 1
        histogram[bin_index] += 1
            
    total_voxels = len(normalized_data)
    pdf = [count / total_voxels for count in histogram]
    
    # Step 2: Compute CDF
    # Formula: CDF(x_i) = sum_{x_j < x_i} PDF(x_j)
    cdf = [0.0] * num_bins
    current_sum = 0.0
    for i in range(num_bins):
        cdf[i] = current_sum
        current_sum += pdf[i]
            
    # Step 3: Create Lookup Table
    lut = cdf[:]
    return lut

def apply_equalization(volume_data, lut):
    min_val = min(volume_data)
    max_val = max(volume_data)
    normalized_data = [(v - min_val) / (max_val - min_val) for v in volume_data]
    
    result = []
    num_bins = len(lut)
    for val in normalized_data:
        bin_index = int(val * (num_bins - 1))
        if bin_index >= num_bins:
            bin_index = num_bins - 1
        result.append(lut[bin_index])
    return result
```

**Parameters:**
*   `num_bins`: The resolution of the histogram and LUT. Typical values are not given in the sources.

**Edge Cases and Pitfalls:**
*   **Continuity Breaking:** The method tends to break the continuities of the scalar range, which can lead to visual artifacts [S1, unofficial].
*   **User Control:** It provides very little control over the redistribution process [S1, unofficial].
*   **Precision:** Discrete implementation requires careful binning to avoid precision errors at boundaries [G].

## Subtopics

### [Range Mapping](5650-range-mapping.md)
Range mapping is a generalization of histogram equalization that uses piecewise linear functions to provide user control while maintaining continuity. Choose this when user-defined feature emphasis is required.

### Histogram Stretching with Percentile Clipping
Histogram Stretching with Percentile Clipping is a post-processing step to create a balanced RGB image by suppressing high intensity outliers caused by constructive interference phenomena inherent to diffraction [S2]. Percentile clipping is applied to suppress high intensity outliers, where the maximum intensity $I_{max}$ per color channel is determined by the maximum intensity value after clipping the higher values belonging to the high 0.15 percentile [S2]. Histogram stretching modifies the intensity range using the formula: $I(x, y) = \frac{I(x, y) - I_{min}}{I_{max} - I_{min}}$, where $I_{min}$ and $I_{max}$ are the minimum and maximum values of the intensity respectively [S2].

## Operating characteristics
Due to its speed and simplicity, it is common to use a discrete histogram equalization and perform this mapping with a lookup table [S1, unofficial]. The time complexity is not given in the sources. The memory complexity is not given in the sources. Strengths include speed and simplicity, and it is effective at spreading out clustered regions automatically [S1, unofficial]. Limitations include a lack of user control, breaking scalar range continuity, and the automatic nature may not align with semantic features [S1, unofficial]. It should be used when automatic enhancement of HDR data is needed and user interaction is not desired [S1, unofficial]. It should not be used when continuity of the scalar range is critical or when specific regions need targeted emphasis via user control [S1, unofficial].

## Background
Histogram Equalization was introduced in the context of interactive transfer function specification for direct volume rendering, specifically to address the difficulties of designing transfer functions for high dynamic range (HDR) data [S1, unofficial]. It serves as the basis for the more controlled Range Mapping technique [S1, unofficial].

## Sources
[S1] PDF Interactive Transfer Function Specification for Direct Volume Rendering, https://www.sci.utah.edu/~stevec/papers/transfer_functions.pdf, fetched 2026-10-28 (unofficial)
[S2] Three-dimensional rendering of computer-generated holograms acquired from point-clouds on light field displays, https://doi.org/10.1117/12.2237581, fetched 2026-11-04
