# Improved Order Independent Transparency for Opaque Kernels


## Overview
This method addresses the transparency artifacts inherent in standard Order Independent Transparency (OIT) when rendering opaque or semi-opaque kernels, such as those used in volumetric spot noise. While standard OIT approximates the correct blending order by accumulating fragments without depth sorting, it fails to correctly represent the visual weight of the closest fragment, leading to incorrect color mixing in close-up views [S1, unofficial]. This approach improves upon standard OIT by explicitly incorporating the contribution of the closest kernel sample into the final color calculation, thereby reducing transparency artifacts while maintaining performance comparable to geometry instancing [S1, unofficial]. It sits between basic OIT and geometry instancing, offering a balance between artifact reduction and rendering speed.

## How it works
The method combines a depth-aware pass for the closest sample with an OIT-based accumulation for the remaining samples.

### Inputs and Outputs
- **Input**: A set of kernel clusters $\mathcal{C}_i$ (or slices), where each cluster contains multiple kernel samples. Each sample $\mathbf{p}_j$ has a position, a transparency $\alpha_j$ (or $T(\mathbf{p}_j)$), and contributes a color $L_{\mathcal{C}_i}$. The direction $\mathbf{d}$ is the view direction.
- **Output**: The final pixel color $L$.

### Data Structures
The implementation relies on standard frame buffers used in OIT:
1.  **Depth Buffer**: To identify the closest kernel sample $\mathbf{p}_f$.
2.  **Accumulation Buffers**: Standard OIT buffers (e.g., linked lists, atomic counters, or weighted blending buffers) to store the sum of color contributions $\sum L_{\mathcal{C}_i}$ and the sum of transparencies $\sum T(\mathbf{p}_j)$ for all fragments. The exact structure depends on the underlying OIT variant (e.g., Weighted Blended OIT or Linked List OIT), but the key requirement is the ability to compute the weighted average of the "rest" of the fragments.

### Steps
1.  **Closest Sample Identification**: Perform a depth test to identify the closest kernel sample $\mathbf{p}_f$ for the current pixel. Retrieve its transparency $\alpha_f$ and the kernel value $\tilde{K}(\mathbf{d}, \mathbf{p}_f)$ at the view direction $\mathbf{d}$. [S1, unofficial]
2.  **Direct Contribution Calculation**: Calculate the direct contribution of the closest kernel:
    $$ C_{direct} = \tilde{K}(\mathbf{d}, \mathbf{p}_f) \alpha_f $$
3.  **OIT Accumulation**: Render all kernel samples using the standard OIT accumulation pass. This pass computes two sums for the pixel:
    -   $S_L = \sum_{i=0}^{N-1} L_{\mathcal{C}_i}$: The sum of color contributions from all clusters/samples.
    -   $S_T = \sum_{i=0}^{N-1} \sum_{\mathbf{p}_j \in \mathcal{C}_i^*} T(\mathbf{p}_j)$: The sum of transparencies from all clusters/samples.
    *Note: In some OIT implementations, these sums might be weighted or stored in specific buffers. The core principle is that $S_L$ and $S_T$ represent the aggregate contribution of all fragments, including the closest one.* [S1, unofficial]
4.  **Weighted Average Calculation**: Compute the weighted average color of all kernels:
    $$ C_{avg} = \frac{S_L}{S_T} $$
    *If $S_T$ is zero (no fragments), $C_{avg}$ is undefined or zero.* [G]
5.  **Remaining Opacity Contribution**: Calculate the contribution of the remaining opacity (all kernels except the closest one) using the weighted average:
    $$ C_{remaining} = C_{avg} \times (1 - \alpha_f) $$
6.  **Final Color Composition**: Sum the direct contribution of the closest kernel and the weighted average contribution of the rest:
    $$ L \approx C_{direct} + C_{remaining} $$
    Substituting the terms:
    $$ L \approx \tilde{K}(\mathbf{d}, \mathbf{p}_f) \alpha_f + \left( \frac{\sum_{i=0}^{N-1} L_{\mathcal{C}_i}}{\sum_{i=0}^{N-1} \sum_{\mathbf{p}_j \in \mathcal{C}_i^*} T(\mathbf{p}_j)} \right) (1 - \alpha_f) $$

### Parameters
-   **N**: The number of kernel clusters or slices. In the source experiments, 32 samples per ray per evaluated kernel were used [S1, unofficial].
-   **$\alpha_f$**: Transparency of the closest kernel.
-   **$T(\mathbf{p}_j)$**: Transparency at sample point $\mathbf{p}_j$.

### Edge Cases and Pitfalls
-   **Zero Transparency Sum**: If $S_T = 0$, the division in step 4 is undefined. This occurs if no kernels are present or all have zero transparency. The result should default to the background color or zero. [G]
-   **Depth Precision**: The identification of $\mathbf{p}_f$ relies on the depth buffer. In dense splatting scenarios, depth precision issues (z-fighting) can lead to incorrect identification of the closest sample. This method mitigates but does not eliminate z-fighting artifacts entirely, unlike geometry instancing which avoids splatting overlap issues but suffers from its own aliasing. [S1, unofficial]
-   **Opaque Kernels**: The method is specifically designed to improve results for opaque kernels. For highly transparent kernels, the results are nearly identical to standard OIT. [S1, unofficial]

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_improved_oit_opaque_kernels(kernels, view_direction):
    """
    Renders kernels using Improved OIT for Opaque Kernels.
    
    Args:
        kernels: List of kernel objects, each with position, transparency, color contribution, and kernel function.
        view_direction: The view direction vector d.
        
    Returns:
        final_color: The computed color L.
    """
    
    # Step 1: Identify the closest kernel sample
    # This typically involves a depth test during rendering or a pre-pass.
    # For the sake of this pseudocode, we assume we have access to the closest kernel.
    closest_kernel = find_closest_kernel(kernels)
    
    if closest_kernel is None:
        return background_color
        
    p_f = closest_kernel.position
    alpha_f = closest_kernel.transparency
    K_df = closest_kernel.kernel_function(view_direction, p_f)
    
    # Step 2: Direct contribution of the closest kernel
    C_direct = K_df * alpha_f
    
    # Step 3: OIT Accumulation for all kernels
    # This step aggregates contributions from ALL kernels, including the closest one.
    sum_L = 0.0
    sum_T = 0.0
    
    for kernel in kernels:
        # Calculate contribution of each kernel
        # L_C_i is the color contribution from cluster i (or kernel i)
        # T(p_j) is the transparency at sample point p_j
        L_i = kernel.color_contribution
        T_j = kernel.transparency
        
        sum_L += L_i
        sum_T += T_j
        
    # Step 4: Weighted average color of all kernels
    if sum_T == 0:
        C_avg = 0.0
    else:
        C_avg = sum_L / sum_T
        
    # Step 5: Remaining opacity contribution
    C_remaining = C_avg * (1 - alpha_f)
    
    # Step 6: Final color composition
    final_color = C_direct + C_remaining
    
    return final_color
```

## Subtopics

### Weighted Blended Order-Independent Transparency
A sibling method that uses weighted blending to achieve order independence. It differs by not explicitly separating the closest sample's contribution in the same manner, potentially leading to different artifact profiles for opaque kernels. [Link](8931-weighted-blended-order-independent-transparency.md)

## Operating characteristics
-   **Time Complexity**: The method requires a depth test for the closest sample and an OIT accumulation pass. The complexity is dominated by the OIT accumulation, which is generally $O(N)$ where $N$ is the number of fragments, but with a lower constant factor than full sorting. [S1, unofficial]
-   **Memory Complexity**: Requires standard OIT buffers (color and weight/depth accumulators) plus the depth buffer. No additional complex data structures like linked lists are strictly required if using weighted blending OIT, but the exact memory footprint depends on the underlying OIT implementation. [S1, unofficial]
-   **Performance**: The method runs at roughly half the framerate of geometry instancing. For example, geometry instancing achieves 60fps for 800k kernels and 3fps for 10M kernels on a GeForce 980. The improved OIT method is slower than instancing but faster than traditional EWA splatting with depth sorting. In specific tests, a lichen pattern with ~400k impulses ran at ~1.5 fps, and other examples ran between 5 and 60+ fps depending on density. [S1, unofficial]
-   **Strengths**:
    -   Reduces transparency artifacts for opaque kernels compared to standard OIT.
    -   Avoids the severe aliasing artifacts (z-fighting) of geometry instancing.
    -   Handles semi-transparency.
    -   Performance is close to geometry instancing, making it suitable for real-time applications. [S1, unofficial]
-   **Limitations**:
    -   Slower than geometry instancing.
    -   Still susceptible to some depth-related artifacts if depth precision is limited.
    -   Not as precise as full depth-sorted EWA splatting for distinguishing neighboring kernels. [S1, unofficial]
-   **When to Use**: Use this method when rendering opaque or semi-opaque kernels (like spot noise) where standard OIT produces visible artifacts, but geometry instancing produces too much aliasing, and full depth sorting is too slow. [S1, unofficial]
-   **When Not to Use**: Avoid if maximum performance is required and aliasing is acceptable (use geometry instancing). Avoid if the highest visual fidelity is required and performance is not a constraint (use depth-sorted EWA splatting). [S1, unofficial]

## Background
This method was introduced in the context of volumetric spot noise synthesis for procedural 3D shell textures. It was developed to address the limitations of existing rendering techniques for dense, opaque kernel-based noise models. The work is attributed to the authors of "Volumetric Spot Noise for Procedural 3D Shell Texture Synthesis" [S1, unofficial]. The method builds on the principles of Order Independent Transparency (OIT) and EWA splatting, aiming to provide a practical real-time solution for rendering complex volumetric textures. [S1, unofficial]

## Sources
[S1] Volumetric Spot Noise for Procedural 3D Shell Texture Synthesis, https://hal.science/hal-02413269v1, fetched 2026-10-26 (unofficial)
