# Improved Order Independent Transparency for Opaque Kernels


## Overview
This method improves the rendering of opaque kernels by incorporating the contribution of the closest sample into the transparency accumulation, reducing transparency artifacts compared to standard OIT while maintaining performance close to geometry instancing [S1, unofficial]. It addresses the limitation of standard Order Independent Transparency (OIT) which introduces noticeable transparency artifacts in close-up views because it is not precise enough to distinguish pairs of neighboring kernels [S1, unofficial]. By re-introducing depth information via the closest sample, it avoids these artifacts without the severe performance overhead of depth-sorted EWA splatting [S1, unofficial].

## How it works
The method combines a direct contribution from the single closest kernel with a weighted average of all other kernels, approximating the correct alpha compositing result in a single pass (or minimal passes) by leveraging a depth test to identify the closest sample [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A set of kernel clusters $\mathcal{C}_i$ (or slices), where each cluster contains sample points $\mathbf{p}_j$. For each pixel, the method requires the color contribution $L_{\mathcal{C}_i}$ and transparency $T(\mathbf{p}_j)$ for all samples, and the specific data for the closest kernel sample $\mathbf{p}_f$.
*   **Output:** The final pixel color $L$.

**Data Structures:**
*   **Closest Sample Data:** The sample point $\mathbf{p}_f$ and its transparency $\alpha_f$ are identified using a standard depth test [S1, unofficial].
*   **Accumulators:** Standard OIT accumulators for color and weight (transparency) are used to store the sum of contributions from all kernels except the closest one [S1, unofficial].

**Parameters:**
*   $N$: The number of kernel clusters or slices used in the summation. Typical values in experiments are 32 samples per ray per evaluated kernel [S1, unofficial].

**Algorithm Steps:**
1.  **Identify Closest Kernel:** Determine the closest kernel sample $\mathbf{p}_f$ and its transparency $\alpha_f$ using a depth test [S1, unofficial].
2.  **Calculate Direct Contribution:** Compute the direct contribution of the closest kernel: $\tilde{K}(\mathbf{d}, \mathbf{p}_f) \alpha_f$, where $\tilde{K}(\mathbf{d}, \mathbf{p}_f)$ is the kernel value at the closest sample point $\mathbf{p}_f$ with direction $\mathbf{d}$ [S1, unofficial].
3.  **Compute Weighted Average:** Calculate the weighted average color of all other kernels by dividing the sum of their color contributions $\sum L_{\mathcal{C}_i}$ by the sum of their transparencies $\sum T(\mathbf{p}_j)$ [S1, unofficial].
4.  **Scale by Remaining Opacity:** Multiply this average by the remaining opacity $(1 - \alpha_f)$ [S1, unofficial].
5.  **Combine:** Sum the direct contribution and the weighted average contribution to get the final color $L$ [S1, unofficial].

**Formula:**
The final color $L$ is approximated by:

$$L \approx \tilde{K}(\mathbf{d}, \mathbf{p}_f) \alpha_f + \left( \frac{\sum_{i=0}^{N-1} L_{\mathcal{C}_i}}{\sum_{i=0}^{N-1} \sum_{\mathbf{p}_j \in \mathcal{C}_i^*} T(\mathbf{p}_j)} \right) (1 - \alpha_f) \quad (1)$$

Where:
*   $L$ is the final color.
*   $\tilde{K}(\mathbf{d}, \mathbf{p}_f)$ is the kernel value at the closest sample point $\mathbf{p}_f$ with direction $\mathbf{d}$.
*   $\alpha_f$ is the transparency of the closest kernel.
*   $N$ is the number of kernel clusters/slices.
*   $L_{\mathcal{C}_i}$ is the color contribution from cluster $i$.
*   $T(\mathbf{p}_j)$ is the transparency at sample point $\mathbf{p}_j$ in cluster $\mathcal{C}_i^*$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_improved_oit_opaque_kernels(closest_sample, all_kernels):
    """
    Renders the final color using Improved OIT for Opaque Kernels.
    
    Args:
        closest_sample: A structure containing:
            - p_f: The closest sample point.
            - alpha_f: The transparency of the closest sample.
            - kernel_value: The value of the kernel at p_f for the current direction d.
        all_kernels: A list or accumulator of all other kernel contributions.
            - sum_L: Sum of color contributions L_C_i for all i.
            - sum_T: Sum of transparencies T(p_j) for all p_j in all clusters.
            
    Returns:
        L: The final pixel color.
    """
    # Step 1 & 2: Direct contribution of the closest kernel
    # K(d, p_f) is the kernel value, alpha_f is its transparency
    direct_contribution = closest_sample.kernel_value * closest_sample.alpha_f
    
    # Step 3: Compute weighted average color of all other kernels
    # Avoid division by zero if no other kernels contribute
    if all_kernels.sum_T == 0:
        average_color = 0.0
    else:
        average_color = all_kernels.sum_L / all_kernels.sum_T
        
    # Step 4: Multiply average by remaining opacity
    remaining_opacity = 1.0 - closest_sample.alpha_f
    weighted_average_contribution = average_color * remaining_opacity
    
    # Step 5: Sum contributions
    L = direct_contribution + weighted_average_contribution
    
    return L
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If the sum of transparencies for the non-closest kernels is zero, the weighted average term is undefined. The implementation must handle this case, typically by treating the average color as zero or ignoring the term [G].
*   **Opaque vs. Transparent:** The method shows significant improvements for opaque kernels but results are almost identical to standard OIT for highly transparent kernels [S1, unofficial].
*   **Depth Precision:** While it avoids z-fighting artifacts associated with geometry instancing, it relies on a depth test to identify the closest sample, which may be subject to standard depth buffer precision limits [S1, unofficial].

## Subtopics

### Weighted Blended Order-Independent Transparency
*   **What it is:** A sibling method that uses weighted blending to achieve order independence.
*   **Difference:** It does not explicitly separate the closest kernel's contribution in the same manner; it relies on a different weighting scheme for all fragments.
*   **When to choose:** When a more general OIT solution is needed without the specific optimization for opaque kernels.
*   **Link:** [Weighted Blended Order-Independent Transparency](8931-weighted-blended-order-independent-transparency.md)

## Operating characteristics
*   **Time Complexity:** The method runs at roughly half the framerate of geometry instancing and basic OIT methods [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On a GeForce 980 with a 1920×1080 rendering window, interactive examples using flat Gaussian kernels run between 5 and 60+ fps, depending on kernel density and viewpoint [S1, unofficial]. For a lichen pattern with ~400k impulses, it runs at ~1.5 frames per second [S1, unofficial].
*   **Strengths:** Avoids aliasing artifacts (z-fighting) present in geometry instancing; handles semi-transparency; reduces transparency artifacts in close-up views compared to standard OIT [S1, unofficial].
*   **Limitations:** Performance is lower than geometry instancing (which offers 60fps for 800k kernels and 3fps for 10M kernels) [S1, unofficial].
*   **When to use:** When rendering opaque kernels where standard OIT shows artifacts and geometry instancing shows z-fighting, and performance is not the absolute highest priority.
*   **When not:** When maximum performance is required and aliasing artifacts are acceptable, or when kernels are highly transparent (standard OIT is sufficient).

## Background
The method was introduced in the context of Volumetric Spot Noise for Procedural 3D Shell Texture Synthesis [S1, unofficial]. It was developed to address the trade-offs between geometry instancing (fast but aliased, no semi-transparency), traditional EWA splatting (accurate but slow due to sorting), and basic OIT (fast but artifact-prone for opaque kernels) [S1, unofficial]. The work was published in 2013 in the Journal of Computer Graphics Techniques (JCGT) [S1, unofficial].

## Sources
[S1] Volumetric Spot Noise for Procedural 3D Shell Texture Synthesis, https://hal.science/hal-02413269v1, fetched 2026-10-26 (unofficial)
