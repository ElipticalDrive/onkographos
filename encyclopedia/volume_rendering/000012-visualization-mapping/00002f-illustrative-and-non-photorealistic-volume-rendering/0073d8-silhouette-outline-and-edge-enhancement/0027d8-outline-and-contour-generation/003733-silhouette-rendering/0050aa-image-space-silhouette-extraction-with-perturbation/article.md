# Image-space silhouette extraction with perturbation

Also called: Image-Space Silhouette Detection, Image-space silhouette extraction, Silhouette effect

## Overview
This method extracts surface silhouettes in image space using edge detection filters, optionally modified by a procedural perturbation function for artistic variation, and combines them with Direct Volume Rendering (DVR) for depth perception [S1, unofficial]. It is chosen for interactive frame rates with huge or animated data sets, as it avoids the high computational effort required by object-space and hybrid algorithms that rely on processing large integral surface meshes [S2].

## How it works
**Inputs:**
1. A volume dataset or pre-rendered DVR image $I_{\text{dvr}}(x, y)$ [S1, unofficial].
2. An edge detection filter kernel (e.g., Sobel, Canny) [G].
3. A procedural perturbation function definition $P(x, y)$ [G].

**Outputs:**
1. A final composite image containing the DVR result overlaid with extracted silhouettes/feature lines [S1, unofficial].

**Data Structures:**
* **Rendered Image:** A 2D array of color values (RGBA) corresponding to the pixel grid [S1, unofficial].
* **Edge Map:** A 2D array of scalar values (or binary mask) representing edge strength or presence [S1, unofficial].
* **Perturbation Field:** A 2D array of scalar values representing the perturbation magnitude or direction at each pixel [G].

**Steps:**
1. **Render Volume:** Render the volume using Direct Volume Rendering (DVR) to produce the base image $I_{\text{dvr}}(x, y)$ [S1, unofficial].
2. **Apply Edge Detection:** Apply an image processing filter (edge detector) to $I_{\text{dvr}}$ to extract feature lines, hatching strokes, ridges, and valley lines [S1, unofficial]. Let the output be the edge map $E(x, y)$ [S1, unofficial].
3. **Apply Perturbation (Optional):** Modify the edge map $E(x, y)$ using a procedural perturbation function $P(x, y)$ to obtain the perturbed edge map $E'(x, y)$ [S1, unofficial].
   * Formula: $E'(x, y) = \text{Modify}(E(x, y), P(x, y))$ [G].
   * Note: The exact mathematical form of $\text{Modify}$ and $P$ is not given in the sources [G]. A common implementation might involve displacing edge pixels or modulating edge strength based on noise fields [G].
4. **Composite:** Combine the extracted silhouettes/feature lines ($E'$) with the DVR result ($I_{\text{dvr}}$) to enhance main features while preserving contextual information [S1, unofficial].
   * Formula: $I_{\text{final}}(x, y) = \text{Composite}(I_{\text{dvr}}(x, y), E'(x, y))$ [G].
   * Note: The specific compositing operator (e.g., alpha blending, max, overlay) is not defined in the sources [G]. Typically, silhouettes are rendered with high opacity over the volume [G].

**Parameters:**
* Edge detection threshold: Not given in the sources. [G]
* Perturbation intensity: Not given in the sources. [G]
* Filter kernel size: Not given in the sources. [G]

**Edge Cases and Pitfalls:**
* **Precision:** Edge detection on noisy volume renderings may produce spurious edges [G].
* **Boundaries:** Silhouettes at the image border may be truncated [G].
* **Degenerate Input:** Uniform regions will yield no edges [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_silhouette_with_perturbation(I_dvr, edge_kernel, P):
    # Step 1: I_dvr is the base DVR image [S1, unofficial]
    # Step 2: Apply edge detection filter to I_dvr
    E = apply_edge_filter(I_dvr, edge_kernel) # Standard 2D convolution [G]
    # Step 3: Apply procedural perturbation to edge map
    # Exact modification formula not given in sources [G]
    if P is not None:
        E_prime = apply_perturbation(E, P) # Displace edge pixels or modulate strength based on noise [G]
    else:
        E_prime = E
    # Step 4: Composite perturbed edges over DVR result
    # Exact compositing operator not given in sources [G]
    I_final = composite(I_dvr, E_prime) # Typically alpha blending or max opacity [G]
    return I_final
```

## Subtopics

### Brush Path Silhouette Tracking
A method by Kalnins et al. using brush path parameterization to represent silhouettes, tracked between frames using image space local search [S4]. It provides frame coherence through the tracking [S4].

## Operating characteristics
* **Time Complexity:** Dominated by DVR rendering and 2D image processing filters [S1, unofficial][S2]. Generally $O(N)$ where $N$ is the number of pixels, assuming fixed-size kernels [G].
* **Memory Complexity:** $O(N)$ for storing the rendered image and edge maps [G].
* **Measured Performance:** Preferred for interactive frame rates with huge or animated data sets [S2].
* **Strengths:** Avoids the high effort of processing large integral surface meshes required by object-space and hybrid algorithms [S2]. Suitable for huge or animated data sets [S2].
* **Limitations:** Relies on the quality of the initial DVR rendering; silhouettes are only as good as the rendered image's contrast [G].
* **When to use:** When interactive frame rates are required for large/animated data, and artistic variation is desired [S1, unofficial][S2].

## Background
The technique is part of the broader field of illustrative and non-photorealistic volume rendering [S1, unofficial]. It builds on the idea of combining traditional illustration effects (like hatching or edge lines) with volume rendering to enhance the perception of salient features [S1, unofficial]. The approach by Yuan and Chen is cited as behaving similarly, using feature lines, hatching strokes, ridges, and valley lines combined with visualization algorithms [S1, unofficial]. Isenberg et al. recommend image-space techniques for interactive frame rates with huge or animated data sets, excluding object-space and hybrid algorithms due to the high effort required for large integral surface meshes [S2].

## Sources
[S1] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
[S2] IRIS: Illustrative Rendering for Integral Surfaces, https://doi.org/10.1109/tvcg.2010.173, fetched 2026-10-27
[S3] Ghosted Illustration Rendering using Depth-based Blending Techniques, https://doi.org/10.15701/kcgs.2020.26.3.1, fetched 2026-10-09
[S4] State of the Art Non-Photorealistic Rendering (NPR) Techniques, https://doi.org/10.2312/localchapterevents/tpcg/tpcg06/089-098, fetched 2026-10-26
