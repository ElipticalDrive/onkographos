# Spectrum and Edge-based Gaussian Splat Count Estimation

Also called: Geometric complexity measurement for Gaussian splats, SA-GS splat count algorithm

## Overview
This method determines the expected number of 3D Gaussian splats per semantic group by analyzing image spectral complexity and edge energy to control geometric detail [S1, unofficial]. It differs from naive frequency magnitude statistics by using a high-pass threshold strategy (implemented via Canny edge detection) to isolate high-frequency signals, which directly dictate Gaussian refinement [S1, unofficial]. This approach outperforms naive mean magnitude because it suppresses dominant low-frequency noise and targets the high-frequency geometric details that require denser Gaussian representation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of masked images $F(x,y)$ belonging to specific semantic groups.
*   **Output:** The expected number of Gaussian splats (or density) for each semantic group, derived from edge counts and geometric aspect ratios.

**Data Structures:**
*   **Image Mask:** Binary or probability mask $F(x,y)$ isolating the semantic region of interest.
*   **Edge Count Map:** $e_{ij}$ representing the count of edge pixels for image $i$ in semantic group $j$.

**Steps:**
1.  **Masked Image Selection:** Identify the subset of training images $M$ and their corresponding masks for a specific semantic group $j$.
2.  **Fourier Transform:** For each masked image $F(x,y)$, compute the 2D Fourier transform to obtain the frequency domain representation $F(u,v)$ [S1, unofficial].
3.  **Polar Conversion:** Convert the Cartesian frequency domain data $F(u,v)$ into polar coordinates $f(\rho, \theta)$, where $\rho$ is the radial frequency and $\theta$ is the angle [S1, unofficial].
4.  **Edge Extraction (High-Pass Approximation):**
    *   Theoretically, apply a high-pass filter using a Dirac delta function $D(\rho) = \delta(T - \rho)$ to isolate high-frequency signals above a threshold $T$ [S1, unofficial].
    *   **Implementation:** Replace the ideal high-pass filter with Canny edge detection. This explicitly sets a threshold for edge selection and extracts edge energy, denoted as $e_{ij}$ for image $i$ in group $j$ [S1, unofficial].
5.  **Perplexity Calculation:** Calculate the total perplexity $P_j$ for semantic group $j$ by summing the edge counts across all images $i$ in the set $M$:
    $$ P_j = \sum_{i \in M} e_{ij} $$
    [S1, unofficial]
6.  **Expected Splat Density:** Determine the expected splat density per pixel $p_i$ in group $j$ based on the Gaussian aspect ratios $a_1 = s_x/s_z$ and $a_2 = s_y/s_z$ (where $s_x, s_y, s_z$ are scales along x, y, z axes) [S1, unofficial]:
    $$ \frac{1}{p_i} = k_1 a_1 $$
    $$ \frac{1}{p_i} = k_2 a_2 $$
    With the constraint $k_1 > k_2$ [S1, unofficial].
7.  **Constant Determination:** The scaling constants $k_1$ and $k_2$ are determined by the overlapping ratio of the input images. Higher overlap requires smaller constants [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def estimate_splat_count(masked_images, semantic_group_id, overlap_ratio):
    """
    Estimates the expected number of Gaussian splats for a semantic group
    based on edge energy and spectral complexity.
    
    Args:
        masked_images: List of tuples (image_mask, image_index) for the group.
        semantic_group_id: Identifier for the semantic group.
        overlap_ratio: Float representing the average overlap of input images.
        
    Returns:
        expected_splat_density: Float, expected splats per pixel.
        perplexity: Float, total edge count for the group.
    """
    total_edge_count = 0.0
    
    for mask, img_idx in masked_images:
        # Step 2 & 3: Fourier Transform and Polar Conversion (Conceptual)
        # In practice, we skip explicit polar integration and use Canny Edge
        # which approximates the high-pass filter D(rho) = delta(T - rho)
        
        # Step 4: Edge Extraction
        # Canny edge detection replaces the ideal high-pass filter
        # to explicitly set a threshold for edge selection.
        edges = canny_edge_detection(mask)
        
        # Count non-zero edge pixels
        edge_count = count_nonzero(edges)
        
        # Step 5: Accumulate Perplexity
        total_edge_count += edge_count
        
    perplexity = total_edge_count
    
    # Step 6 & 7: Calculate Density
    # Constants k1, k2 are derived from overlap_ratio.
    # Higher overlap -> smaller constants.
    # Specific formula for k1/k2 from overlap_ratio is not given in sources [G].
    # Assuming linear scaling or lookup table based on overlap.
    k1 = scale_constant(overlap_ratio, type='high')
    k2 = scale_constant(overlap_ratio, type='low')
    
    # Aspect ratios a1, a2 are typically initialized or estimated from scene geometry.
    # If not provided, default to 1.0 [G].
    a1 = 1.0 
    a2 = 1.0
    
    # Calculate inverse density
    inv_density_1 = k1 * a1
    inv_density_2 = k2 * a2
    
    # Use the more conservative (higher density) estimate or average?
    # Sources define two expectations. We take the minimum p_i (max density)
    # or average. Source implies these are constraints.
    # Let's assume the density is determined by the aspect ratio scaling.
    # p_i = 1 / (k * a)
    
    density_1 = 1.0 / inv_density_1 if inv_density_1 > 0 else 0.0
    density_2 = 1.0 / inv_density_2 if inv_density_2 > 0 else 0.0
    
    # Select the appropriate density metric. 
    # Source lists both. We return the primary one or average.
    expected_splat_density = (density_1 + density_2) / 2.0
    
    return expected_splat_density, perplexity

def scale_constant(overlap_ratio, type):
    """
    Placeholder for determining k1/k2 based on overlap.
    Higher overlap requires smaller constants.
    """
    # Not given in the sources. [G]
    # Example heuristic:
    base = 1.0
    if type == 'high':
        return base * (1.0 / (1.0 + overlap_ratio))
    else:
        return base * (0.5 / (1.0 + overlap_ratio))
```

**Parameters:**
*   $e_{ij}$: Edge count for image $i$ belonging to semantic group $j$ [S1, unofficial].
*   $k_1, k_2$: Scaling constants determined by the overlapping ratio of input images; higher overlap requires smaller constants [S1, unofficial].
*   $T$: Threshold for edge selection in Canny detection (implementation detail) [S1, unofficial].

**Pitfalls:**
*   **Naive Magnitude Failure:** Calculating the mean magnitude of frequency in polar coordinates ($f_\mu = \int_0^{\pi/2} \int_0^\infty \rho \cdot |f(\rho, \theta)| d\rho d\theta$) fails because low-frequency components dominate the integral despite lower weighting, yielding poor complexity estimates [S1, unofficial].
*   **Occlusion Ignorance:** During training, the method constrains the shape of all Gaussians projecting onto the same pixel without explicitly ignoring Gaussians blocked by those with higher opacity values. This may result in all Gaussians conforming to the shape of the semantic group that occupies the largest area, potentially causing artifacts [S1, unofficial].

## Subtopics

### [Semantic-Aware Gaussian Splatting (SA-GS)](19438-semantic-aware-gaussian-splatting-sa-gs.md)
SA-GS is a 3D Gaussian Splatting variant that incorporates semantic masks and geometric complexity constraints to refine Gaussian shapes and reduce the number of splats, thereby improving geometric reconstruction quality and reducing GPU memory consumption. It differs from vanilla 3DGS by adding semantic-aware shape constraints (disk-like for low frequency, stick/dot for high frequency) and alpha-based point extraction instead of using Gaussian means directly. It builds on [Spectrum and Edge-based Gaussian Splat Count Estimation](19432-spectrum-and-edge-based-gaussian-splat-count-estimation.md) by using spectral complexity to determine expected Gaussian counts per semantic group.

## Operating characteristics
*   **Time Complexity:** Dominated by the Canny edge detection and Fourier transforms per masked image. The complexity is $O(N \log N)$ for FFT and $O(N)$ for edge detection, where $N$ is the number of pixels in the mask.
*   **Memory Complexity:** Requires storage for the image masks and edge maps. No large global data structures are introduced beyond the semantic group aggregations.
*   **Strengths:** Effectively isolates high-frequency geometric details that naive frequency statistics miss. Reduces GPU memory consumption by controlling the number of splats per semantic group [S1, unofficial].
*   **Limitations:** The method does not account for occlusion (high-opacity Gaussians blocking others), which can lead to suboptimal shape constraints for overlapping semantic regions [S1, unofficial]. The constants $k_1, k_2$ require heuristic tuning based on overlap ratio [S1, unofficial].
*   **When to Use:** When reconstructing scenes with distinct semantic regions requiring different geometric densities (e.g., fine details vs. smooth surfaces) and GPU memory is a constraint.
*   **When Not to Use:** When occlusion handling is critical for shape accuracy or when semantic segmentation masks are unavailable or unreliable.

## Background
The method originates from the SA-GS (Semantic-Aware Gaussian Splatting) paper, which introduces geometric complexity measurement based on spectrum analysis to dynamically assign expected shapes to Gaussian splats [S1, unofficial]. It was developed to address the "fantasy surface problem" and reduce the number of splats in large scene reconstruction. The approach builds on the observation that high-frequency signals in masked images correlate with the need for denser Gaussian splats.

## Sources
[S1] SA-GS: Semantic-Aware Gaussian Splatting for Large Scene Reconstruction with Geometry Constrain, https://doi.org/10.48550/arXiv.2405.16923, fetched 2026-10-30 (unofficial)
