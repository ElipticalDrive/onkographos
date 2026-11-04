# Mixed-Resolution Multi-Volume Rendering

Also called: Per-channel resolution selection

## Overview
Mixed-resolution multi-volume rendering is a technique for rendering large-scale multi-volume datasets where different channels are rendered at different resolution levels to save memory and reduce sample evaluation [S1, unofficial][S2]. It addresses the problem of high memory bandwidth and computational cost when rendering multi-channel data by exploiting differences in frequency content or importance across channels [S1, unofficial][S2]. What sets it apart from sibling methods like Level-of-detail (LOD) methods, which typically apply a single resolution hierarchy to the entire volume, is the per-channel granularity of resolution selection [S1, unofficial][S2]. It builds on [Resolution Adaptation](3063-resolution-adaptation.md) by extending adaptive resolution to multi-channel datasets.

## How it works
**Inputs and Outputs:**
*   **Input:** A multi-volume dataset with $N$ channels, each having a multi-resolution hierarchy (e.g., octree levels $l_0$ to $l_{max}$). Viewing parameters (camera position, ray direction). Transfer functions.
*   **Output:** A rendered image where each pixel's color is computed using samples from the appropriate resolution level for each channel.

**Data Structures:**
*   **Channel Resolution Constraint Function:** For each channel $c$, a function $f_c(\mathbf{p}, \mathbf{v}, \dots)$ that returns a valid range of resolution levels $[l_{min}, l_{max}]$ or a specific level $l$. The function takes parameters such as:
    *   $\mathbf{p}$: Position in the volume (e.g., ray sample distance to camera).
    *   $\mathbf{v}$: Viewing parameters (e.g., camera orientation, distance).
    *   Frequency content of channel $c$.
    *   Transfer function for channel $c$.
    *   User-defined bounds.
*   **Multi-Resolution Hierarchy:** Each channel has its own hierarchy of resolution levels. Level $l_0$ is the highest resolution, and higher $l$ values indicate lower resolution (coarser grids).

**Numbered Steps:**
1.  **Initialize Ray:** For each pixel, cast a ray through the volume.
2.  **Sample Ray:** For each sample point $\mathbf{p}$ along the ray:
    a.  For each channel $c$:
        i.  Evaluate the constraint function $f_c(\mathbf{p}, \mathbf{v}, \dots)$ to determine the appropriate resolution level $l_c$ for channel $c$ at position $\mathbf{p}$.
        ii. Retrieve the value $V_c(\mathbf{p})$ from the multi-resolution hierarchy of channel $c$ at level $l_c$. This may involve traversing the octree to find the appropriate brick/node.
        iii. Apply the transfer function for channel $c$ to $V_c(\mathbf{p})$ to get opacity $\alpha_c$ and color $C_c$.
    b.  Composite the results from all channels using volume rendering integration (e.g., front-to-back or back-to-front).
3.  **Output:** The final color for the pixel.

**Formulas:**
*   Resolution level selection: $l_c = f_c(\mathbf{p}, \mathbf{v}, \dots)$
*   Volume rendering integration (front-to-back):
    $$ C = \sum_{i=1}^{n} T_i \cdot \alpha_i \cdot C_i $$
    $$ T_i = \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    where $i$ indexes the samples along the ray, and $\alpha_i, C_i$ are the opacity and color from all channels at sample $i$.

**Parameters:**
*   **Resolution Levels:** Defined per channel. Typical values depend on the dataset. For example, in [S1, unofficial], levels $l_0$ to $l_5$ are used.
*   **Constraint Function Parameters:** User-defined or automatically determined based on frequency analysis.

**Edge Cases and Pitfalls:**
*   **High-Frequency Channels:** A channel with high-frequency content may require a higher sampling rate (lower $l$) to avoid aliasing [S1, unofficial][S2].
*   **Low-Importance Channels:** Channels with low importance or low frequency content can be rendered at lower resolutions (higher $l$) to save memory and computation [S1, unofficial][S2].
*   **Boundary Artifacts:** Switching resolution levels abruptly may cause visible artifacts. Smooth transitions or consistent level selection along the ray may be needed.
*   **Memory Management:** Limiting the range of resolutions uploaded to the GPU per channel saves memory [S1, unofficial][S2]. However, managing multiple resolution levels for multiple channels can be complex.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_mixed_resolution_multi_volume(rays, channels, camera_params):
    """
    Render a multi-volume dataset with mixed resolutions per channel.
    
    Args:
        rays: List of rays, each with origin, direction, and t-values.
        channels: List of channel objects, each with:
            - name: str
            - hierarchy: MultiResolutionHierarchy
            - constraint_function: Callable[[Point, CameraParams], int] -> returns resolution level l
            - transfer_function: Callable[[float], Tuple[float, float]] -> returns (alpha, color)
        camera_params: Camera parameters (position, orientation, etc.)
    
    Returns:
        Image: Rendered image.
    """
    image = create_empty_image(rays)
    
    for ray in rays:
        color = (0.0, 0.0, 0.0)
        transmittance = 1.0
        
        for t in ray.t_values:
            point = ray.origin + t * ray.direction
            
            # Accumulate contribution from all channels
            channel_contributions = []
            for channel in channels:
                # Determine resolution level for this channel at this point
                l = channel.constraint_function(point, camera_params)
                
                # Sample the channel at the determined resolution level
                value = channel.hierarchy.sample(point, level=l)
                
                # Apply transfer function
                alpha, c = channel.transfer_function(value)
                channel_contributions.append((alpha, c))
            
            # Composite all channels at this sample point
            # This is a simplified composite; actual implementation may vary
            alpha_total = 0.0
            color_total = (0.0, 0.0, 0.0)
            for alpha, c in channel_contributions:
                alpha_total += alpha * transmittance
                color_total += c * alpha * transmittance
                transmittance *= (1.0 - alpha)
            
            color += color_total
            if transmittance < 1e-6:
                break
        
        image[ray.pixel_index] = color
    
    return image
```

## Subtopics

### Channel-Specific Frequency Analysis
Determines the appropriate resolution level based on the frequency content of each channel. [Link to sibling if exists, otherwise note: Not given in the sources.]

### User-Defined Resolution Bounds
Allows users to manually specify the resolution range for each channel. [Link to sibling if exists, otherwise note: Not given in the sources.]

### View-Dependent Resolution Selection
Adjusts resolution based on the camera's distance and orientation. [Link to sibling if exists, otherwise note: Not given in the sources.]

## Operating characteristics
*   **Time Complexity:** Depends on the number of channels, the resolution levels chosen, and the sampling rate. Generally, rendering time is reduced compared to rendering all channels at the highest resolution [S1, unofficial][S2].
*   **Memory Complexity:** Reduced by limiting the number of resolution levels uploaded to the GPU per channel [S1, unofficial][S2].
*   **Measured Performance:** In [S1, unofficial], rendering human tissue data with 5 channels showed significant memory savings and reduced sample evaluation by using mixed resolutions (e.g., pink channel at $l_3$, orange at $l_5/l_4/l_0$, rest at $l_0$).
*   **Scaling:** Scales well with the number of channels and resolution levels, as each channel can be optimized independently.
*   **Strengths:** Reduces memory usage and computational cost for multi-volume datasets. Allows for flexible resolution selection based on channel importance and frequency content.
*   **Limitations:** May introduce artifacts if resolution levels are not chosen carefully. Complex to implement and manage.
*   **When to Use:** Large-scale multi-volume datasets with varying frequency content or importance across channels.
*   **When Not to Use:** Single-channel volumes or datasets where all channels have similar frequency content and importance.

## Background
Mixed-resolution multi-volume rendering was introduced in the context of scalable web-based volume rendering of large biomedical datasets [S1, unofficial][S2]. It is part of the Residency Octree approach, which uses a hybrid method for managing multi-resolution data in web browsers [S2]. The technique is particularly useful for out-of-core rendering, where memory is a limiting factor [S1, unofficial][S2].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S2] Residency Octree: A Hybrid Approach for Scalable Web ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC10840607/, fetched 2026-10-29
