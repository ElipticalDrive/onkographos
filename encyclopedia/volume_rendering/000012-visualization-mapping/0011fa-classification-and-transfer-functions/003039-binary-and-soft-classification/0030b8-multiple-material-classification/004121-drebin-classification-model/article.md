# Drebin Classification Model

Also called: Automated Percentage Classifier Algorithm, Drebin et al. method, Percentage classifier, Probabilistic material classification, Voxel intensity histogram classification

## Overview
The Drebin Classification Model is a probabilistic method for multi-material volume rendering that classifies each voxel into percentages of various constituent materials based on the relationship between the voxel's intensity and the global intensity histogram [S1][S2, unofficial]. It solves the problem of separating overlapping tissue types (e.g., vascular contrast, soft tissue, calcifications) by estimating material composition per voxel, allowing for the visualization of mixed substances where boundaries are defined by density changes rather than sharp intensity thresholds [S1][S2, unofficial]. It sets itself apart from simple threshold-based methods by using Gaussian approximations of histogram peaks to determine the probability of material presence, thereby handling partial volume effects more naturally [S1].

## How it works

### Inputs and Outputs
**Inputs:**
1. A 3D volume dataset with voxel intensities $I$ [S1].
2. A set of $N$ target materials to classify (e.g., air, fat, soft tissue, bone, contrast) [S2, unofficial].
3. User-defined opacity parameters controlling surface shading and clarity [S1].

**Outputs:**
1. For each voxel, a set of material percentages $p_m$ for each material $m \in \{1, \dots, N\}$, such that $\sum p_m = 1$ [S1].
2. A final rendered image (or video loop) generated via ray casting using these percentages [S1].

### Data Structures
* **Intensity Histogram:** A global histogram of all voxel intensities in the volume [S1].
* **Identify Peaks:** Identify peaks in the histogram [S1].
* **Trapezoid Model:** Each material's intensity distribution is approximated by a trapezoid defined by four parameters:
  * $L_{start}$: Intensity where probability begins to rise.
  * $L_{peak\_start}$: Intensity where probability reaches 100% (plateau start).
  * $L_{peak\_end}$: Intensity where probability begins to fall (plateau end).
  * $L_{end}$: Intensity where probability reaches 0%.
  * *Note:* If the distribution is triangular (no plateau), $L_{peak\_start} = L_{peak\_end}$ [S1].
* **Material Percentage Array:** For each voxel, an array of size $N$ storing the calculated percentage for each material [S1].
* **Trapezoid Boundaries:** Implicitly defined by the histogram peaks and widths [S1].

### Steps

1. **Construct Histogram:** Calculate the voxel intensity histogram for the entire data set [S1].
2. **Identify Peaks:** Identify peaks in the histogram [S1].
3. **Model Distributions:** Model the intensity distribution for each material as a Gaussian distribution centered on the identified peak [S1].
4. **Approximate as Trapezoids:** Approximate these Gaussian distributions as trapezoids to define intensity ranges for each material [S1].
    * *Pitfall:* If a material constitutes a small portion of the total volume, its peak may not be discernible in the histogram [S1].
5. **Calculate Probabilities:** For a voxel with intensity $I$, calculate the probability $P_m(I)$ that it belongs to material class $m$ based on its position relative to the trapezoid of that class [S1]. The probability varies between 0% and 100% depending on the intensity's location within the trapezoid boundaries [S1].
6. **Normalize Percentages:** Assume the percentage of each tissue class contained within the voxel is proportional to the calculated probability [S1]. Normalize the probabilities so they sum to 1:
    $$ p_m = \frac{P_m(I)}{\sum_{k=1}^{N} P_k(I)} $$
    [S1]
7. **Operator Review:** Allow a skilled operator to review and adjust the computer-generated classification (e.g., by adjusting trapezoid boundaries or opacity values) to obtain an anatomically reasonable result [S1].
8. **Ray Casting:** For each pixel in the output image, cast a line from the viewer through the data volume [S1].
9. **Composite:** Compute a weighted sum of the estimated material compositions of all voxels along the ray [S1].
10. **Shading:** Calculate surface shading and enhancement based on the local gradient of voxel composition [S1].
11. **Display:** Generate the final image, typically viewed via a video loop of 16-72 precomputed images rotating about a user-defined axis [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def drebin_classification(volume, materials, opacity_params):
    """
    volume: 3D array of voxel intensities
    materials: list of material names
    opacity_params: dict with user-selected opacity values for shading
    
    Returns:
    voxel_materials: 4D array [D, H, W, N] containing percentages for each material
    """
    N = len(materials)
    D, H, W = volume.shape
    
    # Step 1: Construct Histogram
    histogram = compute_histogram(volume)
    
    # Step 2 & 3: Identify Peaks and Model Distributions
    trapezoids = []
    for m in materials:
        if m in histogram.peaks:
            peak_intensity = histogram.peaks[m]
            # Approximate Gaussian as trapezoid [G]
            trapezoid = gaussian_to_trapezoid(peak_intensity, histogram.width)
        else:
            # Pitfall handling: Estimate location based on known materials [G]
            trapezoid = estimate_trapezoid_from_neighbors(m, materials, trapezoids)
        trapezoids.append(trapezoid)
        
    # Step 5 & 6: Calculate Probabilities and Normalize for each voxel
    voxel_materials = [[[ [0.0] * N for _ in range(W)] for _ in range(H)] for _ in range(D)]
    
    for d in range(D):
        for h in range(H):
            for w in range(W):
                I = volume[d, h, w]
                probabilities = []
                
                for m_idx, trapezoid in enumerate(trapezoids):
                    # Calculate probability P_m(I) based on trapezoid height at I
                    prob = trapezoid_probability(I, trapezoid)
                    probabilities.append(prob)
                
                # Normalize
                total_prob = sum(probabilities)
                if total_prob > 0:
                    for m_idx in range(N):
                        voxel_materials[d][h][w][m_idx] = probabilities[m_idx] / total_prob
                else:
                    # Fallback if no material matches (e.g., noise)
                    for m_idx in range(N):
                        voxel_materials[d][h][w][m_idx] = 1.0 / N
                    
    return voxel_materials

def compute_histogram(volume):
    """Bins voxel intensities and finds peak indices [G]."""
    min_val, max_val = float(min(volume.flatten())), float(max(volume.flatten()))
    num_bins = 256
    bin_width = (max_val - min_val) / num_bins
    counts = [0] * num_bins
    for v in volume.flatten():
        idx = int((v - min_val) / bin_width)
        if idx >= num_bins: idx = num_bins - 1
        if idx < 0: idx = 0
        counts[idx] += 1
    # Find peaks: local maxima significantly above neighbors [G]
    peaks = {}
    for i in range(1, num_bins - 1):
        if counts[i] > counts[i-1] and counts[i] > counts[i+1]:
            intensity = min_val + (i + 0.5) * bin_width
            peaks[f"peak_{i}"] = intensity
    return type('Histogram', (), {'counts': counts, 'peaks': peaks, 'width': bin_width})()

def gaussian_to_trapezoid(peak, width):
    """Maps Gaussian peak and standard deviation to trapezoid bounds [G]."""
    sigma = width
    L_start = peak - 2 * sigma
    L_peak_start = peak - 0.5 * sigma
    L_peak_end = peak + 0.5 * sigma
    L_end = peak + 2 * sigma
    return (L_start, L_peak_start, L_peak_end, L_end)

def estimate_trapezoid_from_neighbors(m, materials, existing_trapezoids):
    """Estimates missing trapezoid bounds by interpolating known material bounds [G]."""
    if len(existing_trapezoids) >= 2:
        avg_start = sum(t[0] for t in existing_trapezoids) / len(existing_trapezoids)
        avg_end = sum(t[3] for t in existing_trapezoids) / len(existing_trapezoids)
        return (avg_start - 10, avg_start - 5, avg_end + 5, avg_end + 10)
    return (0, 0, 0, 0)

def trapezoid_probability(I, trapezoid):
    """
    trapezoid: tuple (L_start, L_peak_start, L_peak_end, L_end)
    Returns probability between 0.0 and 1.0 based on trapezoid geometry [G].
    """
    L_start, L_peak_start, L_peak_end, L_end = trapezoid
    
    if I < L_start or I > L_end:
        return 0.0
    elif L_start <= I < L_peak_start:
        # Linear rise
        return (I - L_start) / (L_peak_start - L_start)
    elif L_peak_start <= I <= L_peak_end:
        # Plateau
        return 1.0
    elif L_peak_end < I <= L_end:
        # Linear fall
        return (L_end - I) / (L_end - L_peak_end)
    else:
        return 0.0

def render_volume(volume, voxel_materials, opacity_params):
    """
    Step 8-11: Ray casting and shading
    """
    # Step 8: For each pixel, cast ray through volume
    # Step 9: Compute weighted sum of material compositions along ray
    # Step 10: Calculate surface shading from local gradient of composition
    # Step 11: Generate image/video loop [G]
    pass
```

### Parameters
* **Opacity Values:** User-selected parameters that control the degree of surface shading, image sharpness, and 3D clarity [S1].
* **Trapezoid Boundaries:** Implicitly defined by the histogram peaks and widths [S1].

### Edge Cases and Pitfalls
* **Small Material Volume:** When a material constitutes a small portion of the total volume, its peak may not be discernible in the histogram [S1].
* **Operator Dependency:** The initial classification is computer-generated and requires review and adjustment by a skilled operator to ensure anatomical reasonableness [S1].
* **Opacity Trade-off:** High opacity values can obscure internal details, such as intravascular abnormalities, which are better seen with lower opacity values [S1].

## Operating characteristics

### Time and Memory Complexity
Not given in the sources.

### Measured Performance
Not given in the sources. The method was implemented in IPRender software and used for CT angiography applications [S1]. The rendering is typically viewed via a video loop of 16-72 precomputed images, suggesting offline or semi-offline processing [S1].

### Scaling
Not given in the sources. The probabilistic classification is independent of the viewing angle, allowing for efficient precomputation of material percentages [S1].

### Strengths and Limitations
* **Strengths:**
    * Handles overlapping tissue types and partial volume effects by estimating material percentages [S1].
    * Provides 3D cues through video loops, which are important for transparent renderings [S1].
    * Allows for operator adjustment to refine anatomical accuracy [S1].
* **Limitations:**
    * Requires operator intervention for optimal results [S1].
    * May obscure internal details if opacity is set too high [S1].
    * Relies on the assumption that material intensities form Gaussian-like distributions [S1].

### When to Use
* Use when visualizing mixed materials with overlapping intensity distributions (e.g., CT angiography with contrast, soft tissue, and calcifications) [S1].
* Use when 3D clarity and surface shading are important, and operator review is feasible [S1].

### When Not to Use
Not given in the sources.

## Background

### Origin
The Drebin Classification Model was introduced by Drebin et al. in the context of CT angiography volume rendering [S1][S2, unofficial]. It is described in the paper "CT angiography with volume rendering: imaging findings" by Kuszyk et al., published in the American Journal of Roentgenology (AJR) in 1995 [S1]. The method was implemented in the IPRender software developed by the Advanced Medical Imaging Laboratory [S1].

### History and Development
The method was developed to address the limitations of Maximum Intensity Projection (MIP) and surface rendering in visualizing vascular abnormalities in large vessels like the aorta [S1]. MIP and surface rendering display only a small fraction of the available intravascular data, whereas volume rendering with the Drebin classifier displays data from the entire volume, providing information about vessel depth similar to conventional angiograms [S1].

### Applications
* **CT Angiography:** Particularly useful for diagnosing vascular abnormalities in larger vessels such as the aorta [S1].
* **Liver and Portal Circulation:** Helps surgeons and non-radiologists integrate planar section information into a 3D context [S1].

## Sources

[S1] CT angiography with volume rendering: imaging findings., https://doi.org/10.2214/ajr.165.2.7618574, fetched 2026-10-26
[S2] An Overview of Rendering from Volume Data --- including Surface and Volume Rendering, https://core.ac.uk/download/63614.pdf, fetched 2026-10-30 (unofficial)
