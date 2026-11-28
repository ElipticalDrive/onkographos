# TF adaptation peak selection
Also called: Valid range peak selection
## Overview
This method selects a single peak from a set of candidates detected in an original or $\alpha$-histogram by prioritizing those within a predefined valid range and maximizing a confidence score $c_p$ [S1]. It differs from its parent, [Alpha-histogram Peak Analysis Scheme](12652-alpha-histogram-peak-analysis-scheme.md), by applying this validity and confidence filter, and from siblings like [Partial Range Histogram (PRH)](12656-partial-range-histogram-prh.md) by relying on $c_p$ rather than partial range constraints alone [S1].
## How it works
**Inputs:**
- A set of detected peaks $P = \{p_1, \dots, p_k\}$, each with an apex value $a_i$ and a confidence score $c_i$ [S1].
- A valid range $[r_{\min}, r_{\max}]$ defining the acceptable density interval [S1].

**Outputs:**
- The selected peak $p_{\text{selected}}$ with apex $a_{\text{selected}}$ [S1].

**Data Structures:**
- Peaks stored as records or tuples containing `apex` and `confidence` fields [G].

**Steps:**
1. **Filter Valid Peaks:** Identify the subset $P_{\text{valid}} \subseteq P$ where each apex satisfies $r_{\min} \le a_i \le r_{\max}$ [S1].
2. **Select by Confidence:** If $P_{\text{valid}}$ is not empty, choose the peak $p_j \in P_{\text{valid}}$ with the maximum confidence score: $p_{\text{selected}} = \arg\max_{p_i \in P_{\text{valid}}} c_i$ [S1].
3. **Fallback Selection:** If $P_{\text{valid}}$ is empty, choose the peak $p_k \in P$ that minimizes the distance to the valid range: $d_k = \min(|a_k - r_{\min}|, |a_k - r_{\max}|)$, and $p_{\text{selected}} = \arg\min_{p_k \in P} d_k$ [S1]. If multiple peaks share the minimal distance, the source does not specify a tie-breaking rule [G: typically, select the one with higher confidence or the first encountered].

**Parameters:**
- **Valid Range:** Defined by the application or preceding analysis steps [S1].
- **Confidence ($c_p$):** Computed by the parent [Alpha-histogram Peak Analysis Scheme](12652-alpha-histogram-peak-analysis-scheme.md) [S1].
- **Static Peak Apex Predictor:** The source uses a reference predictor at the 98.1% percentile of the full histogram area for validation datasets [S1].

**Pitfalls:**
- **Zero Intensity:** Intensity value 0 is discarded from the histogram before peak detection because it is often orders of magnitude more frequent than other values [S1].
- **No Valid Peaks:** If no peaks fall within the valid range, the fallback to the closest peak may select a suboptimal material representation if the valid range is poorly defined [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_peak(peaks, r_min, r_max):
    """
    Selects the best peak based on validity and confidence.
    
    Args:
        peaks: List of dicts with 'apex' and 'confidence' keys.
        r_min: Lower bound of the valid density range.
        r_max: Upper bound of the valid density range.
        
    Returns:
        The selected peak dict.
    """
    # Step 1: Filter peaks within the valid range
    valid_peaks = [p for p in peaks if r_min <= p['apex'] <= r_max]
    
    # Step 2: Select based on availability
    if valid_peaks:
        # Select peak with highest confidence among valid ones
        return max(valid_peaks, key=lambda p: p['confidence'])
    else:
        # Step 3: Fallback to peak closest to the valid range
        def dist_to_range(apex):
            if apex < r_min:
                return r_min - apex
            elif apex > r_max:
                return apex - r_max
            return 0.0
        return min(peaks, key=lambda p: dist_to_range(p['apex']))
```
## Subtopics

### [Histogram-based Fuzzy Classification](13563-histogram-based-fuzzy-classification.md)
A method that uses the selected peak to define fuzzy classification functions, incorporating gradient information to handle noisy data, unlike this method which only selects the peak.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of detected peaks, as it requires a single pass to filter and a single pass to find the maximum or minimum [G].
- **Memory Complexity:** $O(N)$ to store the list of valid peaks [G].
- **Strengths:** Robust to noise by using confidence scores; handles cases where no perfect peak exists by falling back to the closest valid candidate [S1, G].
- **Limitations:** Relies heavily on the accuracy of the "valid range" definition. If the range is too narrow, the fallback mechanism may be triggered frequently [G].
- **When to use:** When automatic classification requires a single dominant peak from a histogram that may contain multiple local maxima or noise-induced peaks [G].
## Background
Developed for automatic transfer function design in volume rendering, with validation performed on MR AAA angiographies by an experienced radiologist [S1]. It operates as a post-processing step following histogram enhancement and peak detection to reduce manual tuning [S1].
## Sources
[S1] The alpha -histogram: Using Spatial Coherence to Enhance Histograms and Transfer Function Design, https://doi.org/10.2312/vissym/eurovis06/227-234, fetched 2026-10-26
