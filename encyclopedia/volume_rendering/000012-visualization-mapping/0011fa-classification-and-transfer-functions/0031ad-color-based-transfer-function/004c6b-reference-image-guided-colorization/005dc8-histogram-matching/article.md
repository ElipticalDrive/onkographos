# Histogram Matching

Also called: Color Alignment

## Overview
Histogram Matching aligns the color of a target image to a reference image by matching their histogram cumulative distribution functions (CDFs) to mitigate color discrepancies from different sensors [S1].

## How it works
**Inputs:**
*   Reference image $I_m$ [S1].
*   Target image $I_w$ [S1].

**Outputs:**
*   Transformed target image $I'_w$ with colors aligned to $I_m$ [S1].

**Data Structures:**
*   Histogram $h(v_i) = n_i / N$, where $n_i$ is the frequency of color $v_i$ and $N$ is the total number of pixels/voxels [S1].
*   Cumulative Distribution Functions $S_m, S_w$ where $S(v) = \sum_{c \le v} h(c)$ [S1].
*   Mapping table $M$ where $M[v_i] = v_k$ [S1].

**Steps:**
1.  **Compute Histograms:** For each color channel, count the frequency $n_i$ of each color $v_i$ and normalize to obtain $h(v_i) = n_i / N$ [S1].
2.  **Compute CDFs:** Calculate the cumulative distribution function for the reference $S_m$ and target $S_w$ using $S(v) = \sum_{c \le v} h(c)$ [S1].
3.  **Build Mapping:** For each level $i \in [0, 255]$ in the target, find the corresponding level $k$ in the reference such that $S_w(v_i) = S_m(v_k)$ [S1].
4.  **Apply Transformation:** Replace each pixel/voxel color $v_i$ in the target with the mapped color $v_k$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def histogram_match(target, reference, levels=256):
    """
    Matches the histogram of target to reference.
    Args:
        target: 1D array/list of color values (e.g., a single RGB channel)
        reference: 1D array/list of color values
        levels: number of discrete bins (typically 256)
    Returns:
        matched: 1D array/list with colors aligned to reference
    """
    N = len(target)
    # 1. Compute histograms
    h_target = [0] * levels
    h_ref = [0] * levels
    for v in target:
        h_target[v] += 1
    for v in reference:
        h_ref[v] += 1
    # Normalize to probabilities
    h_target = [x / N for x in h_target]
    h_ref = [x / N for x in h_ref]
    # 2. Compute CDFs
    S_target = [0] * levels
    S_ref = [0] * levels
    S_target[0] = h_target[0]
    S_ref[0] = h_ref[0]
    for i in range(1, levels):
        S_target[i] = S_target[i-1] + h_target[i]
        S_ref[i] = S_ref[i-1] + h_ref[i]
    # 3. Build mapping
    mapping = [0] * levels
    for i in range(levels):
        # Find k such that S_ref[k] matches S_target[i]
        k = 0
        while k < levels - 1 and S_ref[k+1] < S_target[i]:
            k += 1
        mapping[i] = k
    # 4. Apply transformation
    matched = [mapping[v] for v in target]
    return matched
```
*Note: For RGB images, this function is applied independently to each color channel [G].*

**Parameters:**
*   `levels`: Number of discrete bins. Default 256 for 8-bit per channel [S1].

**Edge Cases:**
Not given in the sources.

## Subtopics

### Optimal Transport Color Matching
A variant that aligns color distributions across multiple camera views by computing an optimal transport map between color histograms, weighted by person segmentation masks to prioritize subject color accuracy [S2, unofficial]. It generates person segmentation masks to identify regions of interest, computes weighted color histograms, and uses the Python Optimal Transport (POT) library to compute the mapping [S2, unofficial]. This method is chosen when specific subjects require higher color fidelity than the background, addressing issues where feature-based matching selects inconsistent colors for the same 3D point across views [S2, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Originated in the context of dual-camera all-in-focus neural radiance fields to mitigate color discrepancies between main and ultra-wide camera sensors [S1]. The method aligns the color of the ultra-wide image to the main image by matching their histogram cumulative distribution functions [S1].

## Sources
[S1] Dual-Camera All-in-Focus Neural Radiance Fields, https://doi.org/10.1109/tpami.2025.3537178, fetched 2026-10-30
[S2] Slow Motion "Bullet Time" with NeRFs, https://labs.laan.com/blogs/slow-motion-bullet-time-with-nerf-neural-radiance-fields, fetched 2026-10-29 (unofficial)
