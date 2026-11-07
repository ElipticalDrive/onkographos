# Intermixing

Also called: Accumulation-level intermixing

## Overview
Intermixing is a strategy for rendering multi-channel volume data where the illumination model is evaluated separately for each data channel at every ray sample, and the resulting contributions are combined [S1, unofficial]. It solves the problem of correctly integrating lighting effects across distinct volumetric datasets (e.g., different tissue types or simulation states) within a single view [S1, unofficial]. It sets itself apart from image-level intermixing by maintaining depth-aware occlusion handling through accumulation, whereas image-level methods fail to handle occlusions between channels correctly due to missing depth information in intermediate results [S1, unofficial]. It is more flexible than other approaches but computationally more expensive [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ co-registered volume datasets (channels), a ray origin $\mathbf{o}$ and direction $\mathbf{d}$, and an illumination model $L(\cdot)$.
*   **Output:** A final pixel color $C_{final}$.

**Data Structures:**
The method maintains per-ray accumulators for color and opacity, similar to standard front-to-back compositing, but updated per-channel.
*   $C_{acc}$: Accumulated color (RGB or RGBA).
*   $\alpha_{acc}$: Accumulated opacity.
*   $C_{channel}$: Temporary color contribution for the current channel.
*   $\alpha_{channel}$: Temporary opacity contribution for the current channel.

**Numbered Steps:**
1.  Initialize accumulators: $C_{acc} = (0,0,0,0)$, $\alpha_{acc} = 0$.
2.  For each ray sample $i$ along the ray:
    a.  For each channel $c$ from $1$ to $N$:
        i.  Sample the volume value $v_c$ at the current position for channel $c$.
        ii. Compute the local color $c_c$ and opacity $\alpha_c$ using the transfer function for channel $c$.
        iii. Evaluate the illumination model $L_c$ using $v_c$ (and potentially normal estimates derived from channel $c$ or shared geometry) to get the lit color $C_{lit, c}$.
        iv. Compute the contribution of channel $c$ using front-to-back compositing relative to the current accumulated state:
            $$ C_{channel} = C_{lit, c} \cdot \alpha_c \cdot (1 - \alpha_{acc}) $$
            $$ \alpha_{channel} = \alpha_c \cdot (1 - \alpha_{acc}) $$
        v. Update the global accumulators:
            $$ C_{acc} \leftarrow C_{acc} + C_{channel} $$
            $$ \alpha_{acc} \leftarrow \alpha_{acc} + \alpha_{channel} $$
    b.  If $\alpha_{acc} \geq 1 - \epsilon$, terminate ray (early ray termination).
3.  Return $C_{acc}$.

**Parameters:**
*   $N$: Number of channels. Implementation-defined, often limited by hardware texture formats (e.g., $N \le 4$ for standard GPU textures) [S1, unofficial].
*   $\epsilon$: Threshold for early ray termination.

**Edge Cases and Pitfalls:**
*   **Occlusion:** Unlike image-level intermixing, this method correctly handles occlusions between channels because the depth/opacity state is shared and updated incrementally [S1, unofficial].
*   **Performance:** Evaluating the illumination model $N$ times per sample makes it computationally expensive compared to single-channel rendering [S1, unofficial].
*   **Homogeneity Optimization:** In octree-based implementations, a node can only be marked homogeneous if *all* corresponding bricks in *all* channels are homogeneous [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def intermixing_ray_cast(ray_origin, ray_direction, volumes, transfer_functions, illumination_model):
    # volumes: list of N volume accessors
    # transfer_functions: list of N TF functions
    # illumination_model: function taking (value, normal) -> color
    
    acc_color = [0.0, 0.0, 0.0, 0.0]
    acc_alpha = 0.0
    
    t = 0.0
    step_size = 0.1 # Example step size
    
    while acc_alpha < 0.99:
        # Sample all channels at current position
        channel_contribs = []
        for c_idx in range(len(volumes)):
            val = volumes[c_idx].sample(ray_origin + ray_direction * t)
            tf = transfer_functions[c_idx]
            local_color, local_alpha = tf(val)
            
            # Evaluate illumination for this channel
            # Note: Normal estimation might be channel-specific or shared
            lit_color = illumination_model(val, estimate_normal(volumes, c_idx, ray_origin + ray_direction * t))
            
            channel_contribs.append((lit_color, local_alpha))
            
        # Composite all channels into the accumulator
        for lit_color, local_alpha in channel_contribs:
            # Front-to-back blending against current accumulator
            weight = local_alpha * (1.0 - acc_alpha)
            acc_color = [
                acc_color[0] + lit_color[0] * weight,
                acc_color[1] + lit_color[1] * weight,
                acc_color[2] + lit_color[2] * weight,
                acc_color[3] + lit_color[3] * weight # Assuming lit_color is RGB, alpha handled separately
            ]
            acc_alpha += local_alpha * (1.0 - acc_alpha)
            
        t += step_size
        
    return acc_color
```

## Subtopics

### [Multi-volume rendering](6407-multi-volume-rendering.md)
Multi-volume rendering refers to the simultaneous rendering of multiple co-registered or deformed volume datasets. It solves the problem of visualizing complex interactions between distinct volumetric data sources (e.g., different tissue types, simulation states, or anatomical atlases) within a single unified view. Unlike single-volume rendering, it requires specific strategies for combining data from multiple textures, categorized into classification-level, accumulation-level, and image-level intermixing. It builds on [Intermixing](5139-intermixing.md) by providing the structural mechanisms to handle multiple data channels during the ray-casting process.

## Operating characteristics
*   **Time Complexity:** $O(S \cdot N)$, where $S$ is the number of samples per ray and $N$ is the number of channels. Each sample requires $N$ evaluations of the illumination model [S1, unofficial].
*   **Memory Complexity:** Depends on the storage of $N$ volume datasets. In octree-based implementations, nodes store metadata for up to $N$ channels [S1, unofficial].
*   **Measured Performance:** Computationally more expensive than single-channel rendering or image-level intermixing due to repeated illumination evaluations [S1, unofficial].
*   **Strengths:** Correctly handles occlusions between different channels; flexible integration of different lighting models per channel [S1, unofficial].
*   **Limitations:** High computational cost; implementation complexity increases with $N$ [S1, unofficial].
*   **When to use:** When accurate occlusion between distinct volumetric channels is required and performance allows [S1, unofficial].
*   **When not to use:** When real-time performance is critical and channels do not occlude each other significantly, or when hardware limits restrict $N$ (e.g., $N > 4$) [S1, unofficial].

## Background
Intermixing, specifically accumulation-level intermixing, was presented by Brix et al. in 2014 for out-of-core rendering of multi-channel volume data sets [S1, unofficial]. It builds on Crassin et al.'s octree-based memory virtualization scheme [S1, unofficial]. The approach is implemented in Voreen [S1, unofficial]. It is also used in Viv [MGP+22] and Neuroglancer [Goo22] [S1, unofficial].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
