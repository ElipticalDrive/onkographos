# Improved Order Independent Transparency for Opaque Kernels


## Overview
This method improves the rendering of opaque kernels by incorporating the contribution of the closest sample into the transparency accumulation, reducing transparency artifacts compared to standard [Order Independent Transparency Approaches](3611-order-independent-transparency-approaches.md) while maintaining performance close to geometry instancing [S1, unofficial]. It addresses the limitation of standard OIT, which introduces noticeable transparency artifacts in close-up views because it is not precise enough to distinguish pairs of neighboring kernels [S1, unofficial]. By re-introducing depth information via the closest sample, it avoids these artifacts without the severe performance overhead of traditional depth-sorted EWA splatting [S1, unofficial].

## How it works
The method combines a geometry instancing pass to identify the closest kernel with an OIT-based EWA splatting pass to handle the remaining semi-transparent contributions.

**Inputs and Outputs:**
*   **Input:** A set of kernel clusters (or slices) $\mathcal{C}_i$, where $i$ ranges from $0$ to $N-1$. Each cluster contains sample points $\mathbf{p}_j$.
*   **Input:** The number of kernel clusters/slices $N$. Typical value: 32 samples per ray per evaluated kernel [S1, unofficial].
*   **Output:** Final color $L$ for each pixel.

**Data Structures:**
*   **Closest Sample:** Identified via a depth test (geometry instancing pass). Let $\mathbf{p}_f$ be the position of the closest kernel sample and $\alpha_f$ be its transparency.
*   **Cluster Contributions:** For each cluster $i$, the color contribution $L_{\mathcal{C}_i}$ and the sum of transparencies of samples within that cluster are accumulated during the OIT pass.

**Steps:**
1.  **Identify Closest Kernel:** Perform a geometry instancing pass to determine the closest kernel sample $\mathbf{p}_f$ for the current pixel using a depth test. Retrieve its transparency $\alpha_f$ and kernel value $\tilde{K}(\mathbf{d}, \mathbf{p}_f)$, where $\mathbf{d}$ is the view direction.
2.  **Accumulate OIT Data:** Perform an Order Independent Transparency pass (e.g., using weighted blending or depth peeling variants from [Order Independent Transparency Approaches](3611-order-independent-transparency-approaches.md)) to compute the total color contribution $\sum_{i=0}^{N-1} L_{\mathcal{C}_i}$ and the total transparency sum $\sum_{i=0}^{N-1} \sum_{\mathbf{p}_j \in \mathcal{C}_i^*} T(\mathbf{p}_j)$ for all *other* kernels. Note: The closest kernel's contribution is excluded from this accumulation to avoid double-counting or self-interference in the average.
3.  **Compute Weighted Average:** Calculate the weighted average color of the remaining kernels by dividing the total color contribution by the total transparency sum.
4.  **Combine Contributions:** Compute the final color $L$ using the formula below.

**Formula:**
$$L \approx \tilde{K}(\mathbf{d}, \mathbf{p}_f) \alpha_f + \left( \frac{\sum_{i=0}^{N-1} L_{\mathcal{C}_i}}{\sum_{i=0}^{N-1} \sum_{\mathbf{p}_j \in \mathcal{C}_i^*} T(\mathbf{p}_j)} \right) (1 - \alpha_f) \quad (1)$$

Where:
*   $L$: Final color.
*   $\tilde{K}(\mathbf{d}, \mathbf{p}_f)$: Kernel value at the closest sample point $\mathbf{p}_f$ with direction $\mathbf{d}$.
*   $\alpha_f$: Transparency of the closest kernel.
*   $N$: Number of kernel clusters/slices.
*   $L_{\mathcal{C}_i}$: Color contribution from cluster $i$.
*   $T(\mathbf{p}_j)$: Transparency at sample point $\mathbf{p}_j$ in cluster $\mathcal{C}_i^*$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_improved_oit_pixel(closest_kernel, oit_accumulator):
    """
    closest_kernel: Object containing p_f (position), alpha_f (transparency),
                    and a function K(d, p) to evaluate kernel value.
    oit_accumulator: Object containing total_color_sum and total_transparency_sum
                     from the OIT pass (excluding the closest kernel).
    """
    # Step 1: Get closest kernel properties
    alpha_f = closest_kernel.alpha_f
    # Evaluate kernel value for the closest sample
    # Note: d is the view direction, typically normalized vector from eye to pixel
    kernel_val = closest_kernel.K(closest_kernel.d, closest_kernel.p_f)
    
    # Step 2 & 3: Get OIT average
    total_color = oit_accumulator.total_color_sum
    total_transparency = oit_accumulator.total_transparency_sum
    
    # Handle edge case: if no other kernels contribute, average is 0
    if total_transparency == 0:
        weighted_avg_color = 0.0
    else:
        weighted_avg_color = total_color / total_transparency
        
    # Step 4: Combine
    # Direct contribution of closest kernel
    direct_contribution = kernel_val * alpha_f
    
    # Weighted average contribution of remaining kernels, scaled by remaining opacity
    remaining_opacity = 1.0 - alpha_f
    average_contribution = weighted_avg_color * remaining_opacity
    
    final_color = direct_contribution + average_contribution
    return final_color
```

**Edge Cases and Pitfalls:**
*   **Zero Transparency Sum:** If the OIT accumulator has zero total transparency (no other kernels), the division in Step 3 must be guarded to avoid NaNs. The average color should be treated as 0 [G].
*   **Opaque Kernels:** This method shows significant improvements for opaque kernels. For highly transparent kernels, results are almost identical to standard OIT [S1, unofficial].
*   **Depth Precision:** While it avoids z-fighting artifacts associated with pure geometry instancing, it still relies on a depth test for the closest sample. Limited depth precision may affect the accuracy of identifying the *true* closest kernel in dense scenes [S1, unofficial].

## Operating characteristics
*   **Performance:** The method runs at roughly half the framerate of geometry instancing or basic OIT [S1, unofficial]. For example, geometry instancing offers 60fps for 800k kernels and 3fps for 10M kernels; this method would be approximately 1.5-30fps for the same counts, though specific benchmarks show ~2 fps for 400k impulses on a GeForce 980 [S1, unofficial].
*   **Hardware:** Tested on OpenGL 4.3 framework with a GeForce 980 [S1, unofficial].
*   **Strengths:** Avoids aliasing artifacts (z-fighting) of geometry instancing; handles semi-transparency; reduces transparency artifacts of standard OIT in close-up views.
*   **Limitations:** Lower performance than geometry instancing or basic OIT; requires two passes (instancing for closest, OIT for rest).
*   **When to use:** When rendering opaque or semi-transparent kernels where visual fidelity (avoiding z-fighting and transparency artifacts) is more important than maximum framerate, and where standard OIT artifacts are unacceptable.
*   **When not to use:** When maximum performance is required and aliasing artifacts are tolerable (use geometry instancing); when kernels are highly transparent and standard OIT artifacts are negligible (use standard OIT).

## Background
This method was introduced in the context of volumetric spot noise for procedural 3D shell texture synthesis [S1, unofficial]. It was developed to address the limitations of previous rendering techniques for noise models composed of flat Gaussian kernels. The approach builds on the principles of Order Independent Transparency and EWA splatting, aiming to balance the performance of geometry instancing with the visual quality of depth-sorted rendering.

## Sources
[S1] Volumetric Spot Noise for Procedural 3D Shell Texture Synthesis, https://hal.science/hal-02413269v1, fetched 2026-10-26 (unofficial)
