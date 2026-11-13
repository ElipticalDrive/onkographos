# Exhaustive Multiple-Peak Search

Also called: Iterative Gaussian Fitting and Subtraction

## Overview
Exhaustive Multiple-Peak Search is an automatic algorithm for detecting all tissue peaks in a global intensity histogram by iteratively identifying the highest peak, fitting a Gaussian, creating a Partial Range Histogram (PRH) for that peak's range, removing the PRH's contribution from the global histogram, and repeating until the histogram is empty [S1]. It sets itself apart from its sibling [Partial Range Histogram (PRH)](12656-partial-range-histogram-prh.md) by providing the iterative search strategy that utilizes PRHs as the core component for isolating distinct materials, and from [TF adaptation peak selection](12678-tf-adaptation-peak-selection.md) by focusing on the detection phase rather than the transfer function construction phase [S1].

## How it works
**Inputs:** A global intensity histogram $H$ of the volume data.
**Outputs:** A set of detected peaks, each represented by a Gaussian approximation $(\mu, \hat{h}, \sigma)$ and an associated PRH.

**Data Structures:**
*   **Global Histogram:** An array of bin counts representing the intensity distribution.
*   **PRH (Partial Range Histogram):** A subset of the global histogram defined by a range $[\mu - \sigma, \mu + \sigma]$, constructed using block selection logic based on range weight $w_r \ge \varepsilon$ [S1].
*   **Peak List:** A collection of detected peaks, each storing $\mu, \hat{h}, \sigma$ and the PRH blocks.

**Parameters:**
*   $\alpha = 1$: The scaling factor for the Gaussian fitting error summation range $[\mu - \alpha\sigma, \mu + \alpha\sigma]$ [S1].
*   $\varepsilon$: The threshold for range weight $w_r$ to include a block in the PRH. Default $\varepsilon = 0.95$ for narrow features, or $\varepsilon = 0.5$ for broader features [S1].
*   Merging Criteria:
    *   Deviation ratio: $\sigma_{max}/\sigma_{min} \le 4$ [S1].
    *   Mean difference: $\mu_{max} - \mu_{min} \le \sigma_{min} \cdot \max(1.0, 2.0 - \sigma_{min}/40.0)$ [S1].

**Steps:**
1.  **Initialize:** Start with the full global histogram $H$.
2.  **Find Peak:** Identify the bin with the maximum count in the current histogram. Let this peak be at intensity $p$.
3.  **Fit Gaussian:** Fit a Gaussian curve $G(x) = \hat{h} e^{-(x-\mu)^2 / (2\sigma^2)}$ to the histogram around $p$. The fitting minimizes the accumulated height difference between the histogram and the middle part of the Gaussian over the range $[\mu - \alpha\sigma, \mu + \alpha\sigma]$ with $\alpha=1$ [S1]. The Gaussian is defined by midpoint $\mu$, height $\hat{h}$, and deviation $\sigma$.
4.  **Create PRH:** Construct a PRH for the range $[\mu - \sigma, \mu + \sigma]$. A block (bin or group of bins) is added to the PRH if its range weight $w_r \ge \varepsilon$. The range weight is defined as $w_r(\Phi, N) = \frac{|N \cap V_\Phi|}{|N|}$, where $N$ is an arbitrary voxel neighborhood and $V_\Phi$ is the set of voxels within range $\Phi$ [S1].
5.  **Remove Contribution:** Remove the voxels belonging to the created PRH blocks from the main histogram. This effectively subtracts the contribution of the detected peak from the global histogram, exposing underlying peaks [S1].
6.  **Repeat:** If the main histogram is not empty, return to Step 2. If a PRH is empty during creation (no blocks satisfy $w_r \ge \varepsilon$), lower the $\varepsilon$ value and retry Step 4 [S1].
7.  **Merge Peaks:** After all peaks are detected, merge similar peaks from the collected PRHs. Two peaks are merged if:
    *   $\frac{\sigma_{max}}{\sigma_{min}} \le 4$
    *   $\mu_{max} - \mu_{min} \le \sigma_{min} \cdot \max(1.0, 2.0 - \frac{\sigma_{min}}{40.0})$
    A new Gaussian is fitted to the merged peak to allow for multiple mergers [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def exhaustive_multiple_peak_search(global_histogram, epsilon_initial=0.95):
    """
    Detects all tissue peaks in a global histogram using iterative PRH creation and subtraction.
    
    Args:
        global_histogram: Array of bin counts.
        epsilon_initial: Initial threshold for range weight.
        
    Returns:
        List of detected peaks, each represented as a dict with keys:
        'mu', 'h', 'sigma', 'prh_blocks'.
    """
    current_histogram = copy(global_histogram)
    detected_peaks = []
    epsilon = epsilon_initial
    
    while histogram_has_peaks(current_histogram):
        # Step 2: Find highest peak
        peak_bin = find_max_bin(current_histogram)
        if peak_bin is None:
            break
            
        # Step 3: Fit Gaussian
        mu, h, sigma = fit_gaussian(current_histogram, peak_bin, alpha=1.0)
        
        # Step 4: Create PRH
        prh_range_start = mu - sigma
        prh_range_end = mu + sigma
        prh_blocks = []
        
        # Iterate over bins in the PRH range
        for bin_idx in range(prh_range_start, prh_range_end + 1):
            # Calculate range weight wr for this bin/block
            # Note: In implementation, N is a neighborhood, V_phi is voxels in range
            # For histogram bins, this simplifies to checking if the bin count
            # is significant relative to the peak or local context.
            # The source defines wr = |N intersect V_phi| / |N|.
            # Here, we assume a simple threshold on the bin height relative to the peak
            # or a local neighborhood weight. The source uses epsilon to filter blocks.
            if is_block_in_prh(current_histogram, bin_idx, mu, sigma, epsilon):
                prh_blocks.append(bin_idx)
                
        if not prh_blocks:
            # If PRH is empty, lower epsilon and retry
            epsilon *= 0.9  # Heuristic to lower threshold
            continue
            
        # Step 5: Remove PRH contribution
        remove_prh_from_histogram(current_histogram, prh_blocks)
        
        # Store peak
        detected_peaks.append({
            'mu': mu,
            'h': h,
            'sigma': sigma,
            'prh_blocks': prh_blocks
        })
        
    # Step 7: Merge similar peaks
    merged_peaks = merge_similar_peaks(detected_peaks)
    return merged_peaks

def is_block_in_prh(histogram, bin_idx, mu, sigma, epsilon):
    """
    Determines if a bin belongs to the PRH based on range weight.
    Simplified implementation: checks if bin height is within a certain
    fraction of the peak height or local neighborhood.
    """
    # The source defines wr = |N intersect V_phi| / |N|.
    # For a 1D histogram, N might be a small window around the bin.
    # V_phi is the set of voxels in the range [mu-sigma, mu+sigma].
    # A simple proxy for wr >= epsilon is checking if the bin is within
    # the range and has a non-negligible height.
    # However, the source emphasizes that PRH does not need to find all blocks,
    # only those with high range weight.
    # We assume a local neighborhood N of size k around bin_idx.
    # V_phi contains all bins in [mu-sigma, mu+sigma].
    # Intersection is bins in N that are also in V_phi.
    
    # For simplicity in this pseudocode, we check if the bin is within
    # the PRH range and its height is above a threshold derived from epsilon.
    # The exact neighborhood calculation is complex and depends on the
    # specific implementation of "voxel neighborhood" in the context of
    # a 1D histogram. The source states: "A block is added to the PRH if
    # the range weight is high enough: wr >= epsilon."
    
    # Heuristic: If the bin is within the range [mu-sigma, mu+sigma]
    # and its height is significant, include it.
    if mu - sigma <= bin_idx <= mu + sigma:
        # Check if the bin is part of the "middle part" of the Gaussian
        # or has high weight. The source uses alpha=1 for fitting,
        # but PRH range is mu +/- sigma.
        # We assume a simple threshold for demonstration.
        return histogram[bin_idx] > 0 # Placeholder for actual wr calculation
    return False

def remove_prh_from_histogram(histogram, prh_blocks):
    """
    Removes the contribution of PRH blocks from the histogram.
    """
    for block in prh_blocks:
        histogram[block] = 0

def merge_similar_peaks(peaks):
    """
    Merges peaks based on deviation ratio and mean difference.
    """
    if not peaks:
        return []
    
    # Sort peaks by mu
    peaks.sort(key=lambda p: p['mu'])
    
    merged = []
    current_peak = peaks[0]
    
    for i in range(1, len(peaks)):
        next_peak = peaks[i]
        
        # Check merging criteria
        sigma_min = min(current_peak['sigma'], next_peak['sigma'])
        sigma_max = max(current_peak['sigma'], next_peak['sigma'])
        mu_diff = abs(next_peak['mu'] - current_peak['mu'])
        
        if sigma_max / sigma_min <= 4 and mu_diff <= sigma_min * max(1.0, 2.0 - sigma_min / 40.0):
            # Merge: fit new Gaussian to combined data
            # This is a simplification; the source says "a new Gaussian is fitted"
            # We assume we combine the PRH blocks and refit.
            combined_blocks = current_peak['prh_blocks'] + next_peak['prh_blocks']
            # Refit Gaussian on the combined data (not implemented in detail here)
            mu_new, h_new, sigma_new = fit_gaussian_combined(combined_blocks)
            current_peak = {
                'mu': mu_new,
                'h': h_new,
                'sigma': sigma_new,
                'prh_blocks': combined_blocks
            }
        else:
            merged.append(current_peak)
            current_peak = next_peak
            
    merged.append(current_peak)
    return merged
```

## Subtopics

### Adaptive Trapezoids
A Transfer Function component (trapezoid) that automatically adapts its center, width, and shape based on the Gaussian approximation $(\mu, \sigma)$ of a detected PRH peak [S1]. It allows users to quickly visualize tissues detected by the PRH algorithm [S1]. The workflow is: 1. Run automatic tissue detection (PRH search). 2. Activate an Adaptive Trapezoid (initially the only TF component). 3. Browse detected PRHs; the trapezoid updates to highlight the current PRH's tissue. 4. Fixate the trapezoid (make it static/disconnected) when an interesting tissue is found. 5. Repeat steps 2-4 for other tissues. 6. Activate all fixed trapezoids and adjust color/opacity [S1]. In manual exploration mode, skip automatic detection and manually slide the center of a partial range across the intensity scale to update the Adaptive Trapezoid [S1].

## Operating characteristics
The algorithm is fully automatic and iterative, with complexity dependent on the number of peaks and the cost of Gaussian fitting and PRH creation [S1]. It is designed to detect all peaks corresponding to distinct materials, even in the presence of overlapping intensity ranges [S1]. The use of PRHs allows for the detection of narrow features, but the merging step ensures that similar peaks are combined [S1]. The algorithm is robust to noise due to the neighborhood-based range weight calculation [S1].

## Background
The method was proposed in 2005 by the authors of "Extending and Simplifying Transfer Function Design in Medical Volume Rendering Using Local Histograms" [S1]. It addresses the problem of automatically detecting tissue peaks in global histograms for medical volume rendering, particularly in cases where tissues have overlapping intensity ranges [S1]. The method builds on the concept of Partial Range Histograms (PRH) and local histogram analysis [S1].

## Sources
[S1] Extending and Simplifying Transfer Function Design in Medical Volume Rendering Using Local Histograms, https://doi.org/10.2312/vissym/eurovis05/263-270, fetched 2026-10-26
