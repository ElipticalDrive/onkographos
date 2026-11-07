# Improved Order Independent Transparency for Opaque Kernels


## Overview
This method improves the rendering of opaque kernels by incorporating the contribution of the closest sample into the transparency accumulation, reducing transparency artifacts compared to standard Order Independent Transparency (OIT) while maintaining performance close to geometry instancing [S1, unofficial]. It addresses the limitation of basic OIT, which fails to distinguish neighboring kernels in close-up views, by re-introducing depth information via the closest sample [S1, unofficial].

## How it works
The method combines a direct contribution from the closest kernel with a weighted average of all other kernels.

**Inputs and Outputs:**
- **Input:** A set of kernel clusters $\mathcal{C}_i$ for $i \in [0, N-1]$, where each cluster contains sample points $\mathbf{p}_j$. For each pixel, the closest kernel sample $\mathbf{p}_f$ and its transparency $\alpha_f$ are identified.
- **Output:** Final color $L$.

**Data Structures:**
- $N$: Number of kernel clusters or slices.
- $\mathcal{C}_i^*$: The set of sample points in cluster $i$.
- $L_{\mathcal{C}_i}$: Color contribution from cluster $i$.
- $T(\mathbf{p}_j)$: Transparency at sample point $\mathbf{p}_j$.
- $\tilde{K}(\mathbf{d}, \mathbf{p}_f)$: Kernel value at the closest sample point $\mathbf{p}_f$ with direction $\mathbf{d}$.

**Steps:**
1. Identify the closest kernel sample $\mathbf{p}_f$ and its transparency $\alpha_f$ using a depth test [S1, unofficial].
2. Calculate the direct contribution of the closest kernel: $\tilde{K}(\mathbf{d}, \mathbf{p}_f) \alpha_f$ [S1, unofficial].
3. Compute the weighted average color of all other kernels by summing their color contributions $\sum L_{\mathcal{C}_i}$ and dividing by the sum of their transparencies $\sum_{i=0}^{N-1} \sum_{\mathbf{p}_j \in \mathcal{C}_i^*} T(\mathbf{p}_j)$ [S1, unofficial].
4. Multiply this average by the remaining opacity $(1 - \alpha_f)$ [S1, unofficial].
5. Sum the direct contribution and the weighted average contribution to get the final color $L$ [S1, unofficial].

**Formula:**
$$L \approx \tilde{K}(\mathbf{d}, \mathbf{p}_f) \alpha_f + \left( \frac{\sum_{i=0}^{N-1} L_{\mathcal{C}_i}}{\sum_{i=0}^{N-1} \sum_{\mathbf{p}_j \in \mathcal{C}_i^*} T(\mathbf{p}_j)} \right) (1 - \alpha_f) \quad (1)$$

Where:
- $L$ is the final color.
- $\tilde{K}(\mathbf{d}, \mathbf{p}_f)$ is the kernel value at the closest sample point $\mathbf{p}_f$ with direction $\mathbf{d}$.
- $\alpha_f$ is the transparency of the closest kernel.
- $N$ is the number of kernel clusters/slices.
- $L_{\mathcal{C}_i}$ is the color contribution from cluster $i$.
- $T(\mathbf{p}_j)$ is the transparency at sample point $\mathbf{p}_j$ in cluster $\mathcal{C}_i^*$.

**Parameters:**
- $N$: The number of kernel clusters or slices. In experiments, 32 samples per ray per evaluated kernel were used [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_color(closest_kernel_value, closest_transparency, cluster_colors, cluster_transparencies):
    """
    Computes the final color using Improved OIT for Opaque Kernels.
    
    Args:
        closest_kernel_value (float): K(d, p_f), the kernel value at the closest sample.
        closest_transparency (float): alpha_f, the transparency of the closest kernel.
        cluster_colors (list of float): List of L_C_i for all clusters.
        cluster_transparencies (list of float): List of sum(T(p_j)) for all clusters.
        
    Returns:
        float: Final color L.
    """
    # Step 2: Direct contribution of the closest kernel
    direct_contribution = closest_kernel_value * closest_transparency
    
    # Step 3: Compute weighted average color of all other kernels
    # Sum of color contributions
    sum_colors = sum(cluster_colors)
    # Sum of transparencies
    sum_transparencies = sum(cluster_transparencies)
    
    # Avoid division by zero if no other kernels contribute
    if sum_transparencies == 0:
        weighted_average = 0.0
    else:
        weighted_average = sum_colors / sum_transparencies
        
    # Step 4: Multiply by remaining opacity
    remaining_opacity_contribution = weighted_average * (1.0 - closest_transparency)
    
    # Step 5: Sum contributions
    final_color = direct_contribution + remaining_opacity_contribution
    
    return final_color
```

**Edge Cases and Pitfalls:**
- **Division by Zero:** If the sum of transparencies for the non-closest kernels is zero, the weighted average is undefined. The pseudocode handles this by setting the average to 0.0 [G].
- **Precision:** The method relies on accurate identification of the closest kernel. Depth precision issues can lead to incorrect identification of $\mathbf{p}_f$, causing artifacts [S1, unofficial].

## Subtopics

## Operating characteristics
- **Time and Memory Complexity:** The method is faster than traditional EWA Splatting because it avoids the severe performance and storage overhead of depth-sorting [S1, unofficial]. It runs at roughly half the framerate of geometry instancing [S1, unofficial].
- **Measured Performance:** On a GeForce 980 with an OpenGL 4.3 framework and a 1920×1080 rendering window, interactive examples using flat Gaussian kernels run between 5 and 60+ fps, depending on kernel density and viewpoint [S1, unofficial]. For a lichen pattern with ≈ 400k impulses, it runs at ≈ 1.5 fps [S1, unofficial].
- **Scaling:** Performance is strongly related to the density and complexity of the kernels [S1, unofficial].
- **Strengths:** Avoids aliasing artifacts (z-fighting) and handles semi-transparency, which geometry instancing cannot [S1, unofficial]. It shows significant improvements for opaque kernels compared to standard OIT [S1, unofficial].
- **Limitations:** Roughly half the framerate of geometry instancing [S1, unofficial]. Basic OIT is not precise enough to distinguish pairs of neighboring kernels, but this method improves upon that [S1, unofficial].
- **When to Use:** When rendering opaque kernels where standard OIT shows transparency artifacts in close-up views, and geometry instancing causes z-fighting [S1, unofficial].

## Background
The method was introduced in the context of Volumetric Spot Noise for Procedural 3D Shell Texture Synthesis [S1, unofficial]. It was developed to address the limitations of existing methods for rendering shell spot noise, specifically the trade-off between performance (geometry instancing) and visual quality (OIT) [S1, unofficial].

## Sources
[S1] Volumetric Spot Noise for Procedural 3D Shell Texture Synthesis, https://hal.science/hal-02413269v1, fetched 2026-10-26 (unofficial)
