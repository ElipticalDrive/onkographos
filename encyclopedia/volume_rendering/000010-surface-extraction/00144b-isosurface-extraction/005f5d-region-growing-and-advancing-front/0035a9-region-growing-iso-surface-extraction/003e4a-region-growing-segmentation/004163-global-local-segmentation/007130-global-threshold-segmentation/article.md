# Global Threshold Segmentation

Also called: Global Segmentation via Intensity Threshold, HMH thresholding, Half-maximum height (HMH) thresholding, Uniform Threshold Segmentation

## Overview
Global threshold segmentation is a technique that applies a single intensity threshold uniformly across all voxels or slices of a dataset to isolate a target structure [S1]. It differs from region-specific methods by using a constant cutoff value for the entire volume, avoiding the computational overhead and manual effort of per-slice adjustments, but failing to account for intra-structure density variations [S1].

## How it works
**Inputs:** A volumetric dataset (e.g., CT scan) with intensity values in Hounsfield Units (HU) or normalized intensity.
**Outputs:** A binary mask or segmented volume where voxels with intensity $I \ge T$ are included in the target structure, and $I < T$ are excluded.

**Data Structures:**
- Input: 3D array $V$ of size $W \times H \times D$, containing float or integer intensity values.
- Output: 3D boolean array $M$ of size $W \times H \times D$.
- Parameter: Scalar threshold $T$.

**Steps:**
1. Determine the global threshold value $T$. This can be done via:
   - **Manual Inspection:** Adjust $T$ iteratively using a Region of Interest (ROI) tool to visually verify the optimal cutoff for the target structure [S1].
   - **Automated HMH (Half-Maximum Height):** Compute the histogram of intensities for the target structure (or the entire volume if the target is dominant). Identify the peak intensity $I_{max}$ and the background minimum $I_{min}$. Calculate $T = \frac{I_{max} + I_{min}}{2}$ [S1]. Note: In some implementations, HMH refers to half the maximum height of the histogram peak relative to the baseline, but the source defines it as determining the cutoff based on half the maximum intensity height of the target structure's histogram or profile [S1].
2. Iterate through every voxel $v$ in $V$.
3. If $V[v] \ge T$, set $M[v] = \text{True}$; otherwise, set $M[v] = \text{False}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def global_threshold_segmentation(volume, threshold_method='manual', manual_threshold=None, histogram_data=None):
    """
    Segments a volume using a single global threshold.
    
    Args:
        volume: 3D numpy array of intensity values.
        threshold_method: 'manual' or 'hmh'.
        manual_threshold: The scalar threshold value if method is 'manual'.
        histogram_data: Intensity histogram data if method is 'hmh'.
        
    Returns:
        mask: 3D boolean array.
    """
    if threshold_method == 'manual':
        if manual_threshold is None:
            raise ValueError("Manual threshold must be provided.")
        T = manual_threshold
    elif threshold_method == 'hmh':
        if histogram_data is None:
            raise ValueError("Histogram data must be provided for HMH.")
        # HMH: Half-maximum height. 
        # Assuming histogram_data provides peak intensity and background level.
        # Source defines it as determining cutoff based on half the maximum intensity height.
        # Implementation detail: Typically T = (I_peak + I_background) / 2 or similar.
        # Since exact formula for "half maximum height" in histogram context is ambiguous,
        # we use the standard interpretation of midpoint between peak and background if not specified.
        # However, source says "half the maximum intensity height of the target structure's histogram".
        # This likely means T = I_max / 2 if background is 0, or (I_max + I_min)/2.
        # We will use the midpoint interpretation as it is standard for "half-max" in signal processing,
        # but note the source's specific phrasing.
        I_max = histogram_data['peak_intensity']
        I_min = histogram_data['background_intensity'] # Assumed 0 or min non-target
        T = (I_max + I_min) / 2.0
    else:
        raise ValueError("Unknown threshold method.")

    # Apply threshold globally
    mask = volume >= T
    return mask
```

**Parameters:**
- **Threshold Range (HU):** For in vivo hyoid bones, the global threshold range is 105–170 HU [S1].
- **Mean Threshold (HU):** 131.10 HU for in vivo hyoid bones [S1].
- **Median/Mode (HU):** Median 129 HU, Mode 120 HU [S1].

**Edge Cases and Pitfalls:**
- **Adjacent Structure Interference:** Applying a threshold derived from an adjacent anatomical structure (e.g., mandible) causes significant portions of the target structure (e.g., hyoid cornua) to be excluded [S1]. The threshold levels for adjacent structures are significantly different (p = 0.04) [S1].
- **Small/Variable Density Structures:** The method fails to accurately model small structures with variable density. Large portions of the target bone are excluded when using both global and per-slice HMH variants [S1].
- **Age Correlation:** Higher thresholds are used when segmenting images from older individuals (R2 = 0.44, p < 0.001) [S1].

## Subtopics

### Region-specific Threshold Segmentation
A segmentation technique that accounts for intra-structure density variations by adjusting intensity thresholds per region or per slice [S1]. It generates the 3D model slice-by-slice, adjusting the threshold value by visual inspection on a per-slice basis and recording the value for each slice [S1]. It uses a wider HU range (85–354 HU) and higher mean/median/mode values (Mean: 173.87 HU, Median: 175 HU, Mode: 146 HU) compared to global thresholding (105–170 HU, Mean: 131.10 HU) [S1]. It is chosen when the target structure has significant density variations that a single global threshold cannot capture, though it requires more manual effort [S1].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(N)$ where $N$ is the number of voxels, as each voxel is compared against a single scalar threshold.
- Memory: $O(N)$ for the output mask.

**Measured Performance:**
- In studies of in vivo hyoid bones, global thresholding resulted in significantly lower volumes compared to region-specific thresholding when the same threshold was not appropriately tuned [S1].
- Using the mandible's threshold for the hyoid bone resulted in significantly lower hyoid volumes (p < 0.001) [S1].

**Strengths and Limitations:**
- **Strengths:** Fully automated (if HMH is used) or simple manual adjustment; computationally efficient; no need for per-slice processing.
- **Limitations:** Fails to model small structures with variable density; excludes thin extensions (e.g., cornua) if the threshold is too high or derived from adjacent structures; less accurate than region-specific methods for heterogeneous structures [S1].

**When to Use/Not Use:**
- Use when the target structure has uniform density and is distinct from the background.
- Do not use for small structures with variable density or when adjacent structures have similar intensities [S1].

## Background
Originating from standard CT segmentation algorithms, this method is often implemented using tools like the "Volume Edit" tool in Analyze software [S1]. The Half-Maximum Height (HMH) technique was introduced by Coleman and Colbert (2007) and Weber and Bookstein (2011) as a fully automated alternative to manual thresholding [S1]. It is used in medical imaging, particularly for 3D reconstruction of anatomical structures like the hyoid bone, where accurate landmark placement and linear measurements are required [S1].

## Sources
[S1] Hyoid bone development: An assessment of optimal CT scanner parameters and 3D volume rendering techniques, https://doi.org/10.1002/ar.23157, fetched 2026-11-08
