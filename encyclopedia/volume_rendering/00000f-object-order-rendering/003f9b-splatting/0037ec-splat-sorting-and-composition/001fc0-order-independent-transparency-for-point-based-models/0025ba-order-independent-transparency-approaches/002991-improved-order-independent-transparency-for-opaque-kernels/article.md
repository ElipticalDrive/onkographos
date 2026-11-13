# Improved Order Independent Transparency for Opaque Kernels


## Overview
This method improves the rendering of opaque kernels by incorporating the contribution of the closest sample into the transparency accumulation, reducing transparency artifacts compared to standard OIT while maintaining performance close to geometry instancing [S1, unofficial]. It addresses the limitation of standard OIT, which introduces noticeable transparency artifacts in close-up views because it is not precise enough to distinguish pairs of neighboring kernels [S1, unofficial]. By re-introducing depth information via the closest sample, it avoids these artifacts while handling semi-transparency, unlike geometry instancing which suffers from z-fighting and cannot handle semi-transparency [S1, unofficial].

## How it works
The method combines a direct contribution from the nearest kernel with a weighted average approximation of the remaining kernels.

**Inputs:**
- The set of kernel clusters/slices $\mathcal{C}_i$ for $i = 0 \dots N-1$.
- For each cluster, the set of sample points $\mathcal{C}_i^*$.
- The direction $\mathbf{d}$ for the current ray/pixel.
- The kernel value function $\tilde{K}(\mathbf{d}, \mathbf{p})$.
- The transparency function $T(\mathbf{p})$.
- The color contribution $L_{\mathcal{C}_i}$ from each cluster.

**Outputs:**
- The final color $L$ for the pixel.

**Data Structures:**
- The implementation requires identifying the closest kernel sample $\mathbf{p}_f$ and its transparency $\alpha_f$. This is typically done using a depth test in the rendering pipeline [S1, unofficial].
- The summation terms $\sum L_{\mathcal{C}_i}$ and $\sum T(\mathbf{p}_j)$ are accumulated during the OIT pass.

**Steps:**
1. Identify the closest kernel sample $\mathbf{p}_f$ and its transparency $\alpha_f$ using a depth test [S1, unofficial].
2. Calculate the direct contribution of the closest kernel: $\tilde{K}(\mathbf{d}, \mathbf{p}_f) \alpha_f$.
3. Compute the weighted average color of all other kernels by dividing the sum of their color contributions $\sum_{i=0}^{N-1} L_{\mathcal{C}_i}$ by the sum of their transparencies $\sum_{i=0}^{N-1} \sum_{\mathbf{p}_j \in \mathcal{C}_i^*} T(\mathbf{p}_j)$.
4. Multiply this average by the remaining opacity $(1 - \alpha_f)$.
5. Sum the direct contribution and the weighted average contribution to get the final color $L$.

**Formula:**
The final color $L$ is calculated as:

$$
L \approx \tilde{K}(\mathbf{d}, \mathbf{p}_f) \alpha_f + \left( \frac{\sum_{i=0}^{N-1} L_{\mathcal{C}_i}}{\sum_{i=0}^{N-1} \sum_{\mathbf{p}_j \in \mathcal{C}_i^*} T(\mathbf{p}_j)} \right) (1 - \alpha_f)
$$

Where:
- $L$ is the final color.
- $\tilde{K}(\mathbf{d}, \mathbf{p}_f)$ is the kernel value at the closest sample point $\mathbf{p}_f$ with direction $\mathbf{d}$.
- $\alpha_f$ is the transparency of the closest kernel.
- $N$ is the number of kernel clusters or slices.
- $L_{\mathcal{C}_i}$ is the color contribution from cluster $i$.
- $T(\mathbf{p}_j)$ is the transparency at sample point $\mathbf{p}_j$ in cluster $\mathcal{C}_i^*$.

**Parameters:**
- $N$: The number of kernel clusters or slices. In the source experiments, 32 samples per ray per evaluated kernel were used [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_improved_oit_color(closest_kernel_value, closest_transparency, 
                               sum_cluster_colors, sum_cluster_transparencies):
    """
    Computes the final color using Improved OIT for Opaque Kernels.
    
    Args:
        closest_kernel_value (float): K(d, p_f), the kernel value at the closest sample.
        closest_transparency (float): alpha_f, the transparency of the closest kernel.
        sum_cluster_colors (float): Sum of L_C_i for all clusters.
        sum_cluster_transparencies (float): Sum of T(p_j) for all samples in all clusters.
        
    Returns:
        float: The final color L.
    """
    # Direct contribution of the closest kernel
    direct_contribution = closest_kernel_value * closest_transparency
    
    # Weighted average of remaining kernels
    # Avoid division by zero if no other kernels contribute transparency
    if sum_cluster_transparencies > 0:
        average_remaining_color = sum_cluster_colors / sum_cluster_transparencies
    else:
        average_remaining_color = 0.0
        
    # Contribution from remaining kernels, modulated by remaining opacity
    remaining_contribution = average_remaining_color * (1.0 - closest_transparency)
    
    # Final color
    final_color = direct_contribution + remaining_contribution
    
    return final_color
```

**Edge Cases and Pitfalls:**
- If `sum_cluster_transparencies` is zero, the division in step 3 is undefined. The pseudocode handles this by defaulting the average remaining color to 0.0 [G].
- The method relies on the accurate identification of the closest kernel $\mathbf{p}_f$. Errors in depth testing can lead to incorrect $\alpha_f$ and $\tilde{K}$ values [S1, unofficial].
- For highly transparent kernels, the results are almost identical to standard OIT, so the complexity is justified primarily for opaque or semi-opaque kernels [S1, unofficial].

## Subtopics

### Weighted Blended Order-Independent Transparency
- **What it is:** A method that assigns weights to fragments based on their depth to achieve order-independent blending.
- **Difference:** It does not explicitly separate the closest kernel; instead, it uses a weighted average of all fragments.
- **When to choose:** When a simpler implementation is preferred and the specific artifacts of opaque kernels are less critical.
- **Link:** [Weighted Blended Order-Independent Transparency](8931-weighted-blended-order-independent-transparency.md)

### Hardware-Z Cull Depth Peeling
- **What it is:** A method that uses the hardware Z-buffer to peel away layers of fragments sequentially.
- **Difference:** It is more computationally expensive as it requires multiple passes but provides exact depth sorting.
- **When to choose:** When exact depth sorting is required and performance is less of a concern.
- **Link:** [Hardware-Z Cull Depth Peeling](13947-hardware-z-cull-depth-peeling.md)

## Operating characteristics
- **Time Complexity:** The method adds a constant overhead to standard OIT by requiring the identification of the closest kernel and the calculation of the direct contribution. The summation terms are accumulated during the standard OIT pass.
- **Memory Complexity:** No additional memory structures are required beyond those used for standard OIT and the depth buffer for identifying the closest kernel [S1, unofficial].
- **Performance:** The method runs at roughly half the framerate of geometry instancing and basic OIT. For example, geometry instancing offers 60fps for 800k kernels, while the improved OIT method runs at approximately 2-3 fps for similar densities (e.g., 400k impulses at 1.5 fps) [S1, unofficial].
- **Strengths:** Avoids aliasing artifacts (z-fighting) present in geometry instancing and handles semi-transparency. It shows significant improvements for opaque kernels compared to standard OIT [S1, unofficial].
- **Limitations:** Lower performance compared to geometry instancing and basic OIT. The approximation may not be as accurate as exact depth sorting methods like depth peeling [S1, unofficial].
- **When to use:** When rendering opaque kernels where standard OIT shows transparency artifacts and geometry instancing suffers from z-fighting or cannot handle semi-transparency.
- **When not to use:** When maximum performance is required and the artifacts of standard OIT or geometry instancing are acceptable.

## Background
The method was introduced in the context of volumetric spot noise for procedural 3D shell texture synthesis [S1, unofficial]. It was developed to address the limitations of existing rendering techniques for opaque kernels, specifically the trade-off between performance and visual quality. The authors noted that while geometry instancing was fast, it introduced aliasing artifacts, and standard OIT, while faster than volume EWA splatting, introduced transparency artifacts in close-up views [S1, unofficial]. The improved OIT method was proposed as a compromise, re-introducing depth information to improve visual quality for opaque kernels while maintaining reasonable performance.

## Sources
[S1] Volumetric Spot Noise for Procedural 3D Shell Texture Synthesis, https://hal.science/hal-02413269v1, fetched 2026-10-26 (unofficial)
