# Intermixing

Also called: Accumulation-level intermixing

## Overview
Intermixing is a technique for rendering multiple co-registered volume channels simultaneously by evaluating illumination models and combining results at different stages of the ray-casting pipeline [S1, unofficial]. It solves the problem of correctly fusing multi-channel data (e.g., RGB color, opacity, or distinct scalar fields) while handling occlusions and depth relationships between channels [S1, unofficial]. It is distinguished from classification-level intermixing by evaluating illumination per channel rather than interpolating input values first, and from image-level intermixing by preserving depth information during accumulation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ co-registered volume datasets (channels), a viewing ray, and an illumination model $I(\cdot)$.
*   **Output:** A single composite color and opacity value for the pixel.

**Data Structures:**
*   **Ray State:** Accumulators for color $C$ and opacity $\alpha$ for each channel. For accumulation-level intermixing, these are maintained separately per channel during the ray march.
*   **Channel Data:** Voxel values $v_c$ for each channel $c \in \{1, \dots, N\}$.

**Numbered Steps:**
1.  **Initialize:** For each channel $c$, initialize accumulated color $C_c = 0$ and accumulated opacity $\alpha_c = 0$.
2.  **Ray Marching:** For each sample point $s$ along the ray:
    a.  **Sample:** Retrieve the voxel value $v_{c,s}$ for each channel $c$ at the current position.
    b.  **Evaluate Illumination:** Compute the local contribution (color and opacity) for each channel independently using the illumination model:
        $$ (C_{local,c}, \alpha_{local,c}) = I(v_{c,s}) $$
        *Note: In classification-level intermixing, step (a) would involve interpolating $v_{1,s}$ and $v_{2,s}$ with a weight $\omega$ before illumination evaluation [S1, unofficial].*
    c.  **Accumulate:** Update the channel-specific accumulators using front-to-back compositing:
        $$ C_c \leftarrow C_c + C_{local,c} \cdot \alpha_{local,c} \cdot (1 - \alpha_c) $$
        $$ \alpha_c \leftarrow \alpha_c + \alpha_{local,c} \cdot (1 - \alpha_c) $$
3.  **Final Compositing:** Combine the accumulated results from all channels. The specific combination formula depends on the implementation, but generally involves blending the channel-specific accumulated colors and opacities. For example, if channels represent color components, they may be summed or blended based on their final opacities.
4.  **Output:** Return the final composite color.

**Parameters:**
*   **Number of Channels ($N$):** Implementation-defined. In GPU-based approaches using standard texture formats, $N$ is often limited to 4 due to RGBA component constraints [S1, unofficial].
*   **Weighting Factor ($\omega$):** Used in classification-level intermixing (not accumulation-level) for linear interpolation between channels, e.g., $\omega = 0.5$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Occlusion Handling:** Image-level intermixing fails to handle occlusions between channels correctly because it renders complete images per channel and mixes them using alpha blending, losing depth information in intermediate results [S1, unofficial]. Accumulation-level intermixing avoids this by maintaining depth-aware accumulation per channel.
*   **Homogeneity Optimization:** In octree-based approaches, nodes can be marked homogeneous if all voxels in corresponding bricks are identical (within a threshold). For multi-channel data, a node is homogeneous only if *all* channels are homogeneous [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def intermixing_ray_cast(ray_origin, ray_direction, volumes, illumination_model):
    """
    Renders a single ray using accumulation-level intermixing.
    
    Args:
        ray_origin: 3D vector
        ray_direction: 3D vector
        volumes: List of volume data structures, one per channel
        illumination_model: Function taking voxel value, returning (color, opacity)
        
    Returns:
        final_color: 3D vector
        final_opacity: float
    """
    num_channels = len(volumes)
    
    # Initialize accumulators for each channel
    # C_c: accumulated color for channel c
    # alpha_c: accumulated opacity for channel c
    C_accum = [vec3(0.0) for _ in range(num_channels)]
    alpha_accum = [0.0 for _ in range(num_channels)]
    
    # Ray marching parameters
    step_size = 0.5 # Example value
    max_steps = 1000
    
    for step in range(max_steps):
        # Calculate current sample position
        t = step * step_size
        pos = ray_origin + ray_direction * t
        
        # Sample and evaluate illumination for each channel
        for c in range(num_channels):
            # Sample voxel value for channel c
            v_c = volumes[c].sample(pos)
            
            # Evaluate illumination model for channel c
            color_local, alpha_local = illumination_model(v_c)
            
            # Front-to-back accumulation for channel c
            # Contribution = color_local * alpha_local * (1 - alpha_accum[c])
            contribution = color_local * alpha_local * (1.0 - alpha_accum[c])
            C_accum[c] = C_accum[c] + contribution
            alpha_accum[c] = alpha_accum[c] + alpha_local * (1.0 - alpha_accum[c])
            
            # Early ray termination check (optional, per channel or global)
            if alpha_accum[c] >= 1.0:
                # Channel c is fully opaque, but other channels might still contribute
                pass

    # Final combination of channel accumulators
    # This step depends on the specific semantics of the channels.
    # For RGB channels, they might be summed or blended.
    # Here we assume a simple weighted blend based on final opacities.
    total_alpha = 0.0
    final_color = vec3(0.0)
    
    for c in range(num_channels):
        # Weight by the channel's final opacity relative to total opacity
        # Or simply sum if channels are additive (e.g., RGB)
        final_color = final_color + C_accum[c]
        total_alpha = total_alpha + alpha_accum[c] # Simplified
        
    # Normalize or blend final color based on total_alpha
    # [G] Specific final blending formula is implementation-dependent.
    # A common approach for RGB is direct summation if alpha is handled separately.
    
    return final_color, total_alpha
```

## Subtopics

### Classification-level intermixing
At each ray sample, the sampled values of two channels are linearly interpolated using a weighting factor $\omega$ (e.g., $\omega = 0.5$) before evaluating an illumination model using the interpolation result [S1, unofficial]. It is less flexible than accumulation-level intermixing but computationally cheaper. Choose when channels represent scalar fields that should be blended before lighting (e.g., mixing two tissue types) [S1, unofficial].

### Image-level intermixing
A complete image is rendered for each channel and the results are mixed using alpha blending [S1, unofficial]. It is computationally efficient but cannot handle occlusions between different channels correctly due to missing depth information in the accumulated intermediate results [S1, unofficial]. Choose when channels are independent and occlusion between them is not a concern [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Accumulation-level intermixing is computationally more expensive than classification-level intermixing because the illumination model is evaluated separately for each channel at each sample point [S1, unofficial].
*   **Memory Complexity:** Depends on the number of channels. In GPU implementations, the number of channels is often limited to 4 due to texture format constraints (RGBA) [S1, unofficial].
*   **Strengths:** Correctly handles occlusions between channels (unlike image-level); flexible combination of illumination models (unlike classification-level) [S1, unofficial].
*   **Limitations:** Higher computational cost; channel count may be hardware-limited [S1, unofficial].
*   **When to use:** When rendering multiple co-registered volumes where inter-channel occlusion matters and different illumination models are needed per channel [S1, unofficial].
*   **When not to use:** When performance is critical and channels can be pre-blended (classification-level) or are independent (image-level) [S1, unofficial].

## Background
Intermixing was introduced in the context of multi-volume rendering, where multiple co-registered volumes are rendered simultaneously [S1, unofficial]. Schubert and Scholl [SS11] provided an overview of combining multiple channels during rendering, differentiating between classification-level, accumulation-level, and image-level intermixing [S1, unofficial]. Accumulation-level intermixing was used by Brix et al. [BPH14] in an out-of-core rendering approach based on Crassin et al.'s [CNLE09] octree-based memory virtualization scheme, implemented in Voreen [DLJL22]. It is also used in Viv [MGP+22] and Neuroglancer [Goo22] [S1, unofficial].

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
