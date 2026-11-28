# Edge Enhancement via Gradient Modulation

Also called: Edge Enhancement, Gradient-based Opacity Modulation

## Overview
Edge Enhancement via Gradient Modulation is a volume rendering technique designed to improve depth perception by disambiguating the depth ordering of overlapping structures, such as blood vessels in medical imaging [S1]. It enhances edges by modulating voxel opacity based on the 3D intensity gradient perpendicular to the viewing direction [S1]. Unlike standard volume rendering which relies on intensity and density, this method specifically targets occlusion cues by highlighting boundaries where the gradient vector is orthogonal to the ray direction [S1].

## How it works
The method operates as a post-processing or fragment shader stage applied during volume ray-marching. It uses the gradient magnitude and direction computed from the volume data (typically via Sobel operators as per [Voxel Gradient Calculation](2102-voxel-gradient-calculation.md)) to determine the final alpha value of a sample.

**Inputs and Outputs:**
*   **Input:** Volume data index, current sample position, ray direction vector ($\vec{r}$), and pre-computed gradient vector ($\vec{g}$) and its alpha component ($\alpha_g$) from the previous step.
*   **Output:** Final RGBA color and alpha value for the current sample.

**Data Structures:**
The implementation relies on standard floating-point vectors.
*   `vec4 sample`: The initial sample from the volume transfer function, containing RGB intensity and alpha.
*   `vec4 n`: The result of `ComputeGradient`, where `n.rgb` is the gradient vector $\vec{g}$ and `n.a` is the gradient magnitude or a mask indicating valid gradient data.
*   `vec2 step`: A pair of floats defining the smoothstep range.

**Parameters:**
*   `sampleThreshold`: Float, default `0.1`. Threshold for the initial sample's alpha to trigger gradient computation [S1].
*   `gradStep`: Float, default `0.0005`. Step size used in the finite difference gradient computation [S1].
*   `stepMin` (`step.x`): Float, default `0.02`. Lower bound for the smoothstep function [S1].
*   `stepMax` (`step.y`): Float, default `0.06`. Upper bound for the smoothstep function [S1].

**Algorithm Steps:**
1.  **Sample Volume:** Retrieve the initial color and alpha from the volume at the current position using the transfer function.
2.  **Check Threshold:** If the sample's alpha is greater than `sampleThreshold`, proceed to gradient analysis; otherwise, retain the original sample.
3.  **Compute Gradient:** Calculate the 3D intensity gradient $\vec{g}$ and its associated alpha/magnitude $\alpha_g$ at the current position. This step assumes the existence of a `ComputeGradient` function that returns a `vec4` where the RGB components are the gradient vector and the A component is the magnitude or validity mask [S1].
4.  **Check Gradient Validity:** If the gradient alpha $\alpha_g$ is greater than 0.0, proceed to modulation; otherwise, retain the original sample.
5.  **Calculate Modulation Factor:** Compute the factor $F$ using the formula:
    $$ F = \|\vec{g}\| \cdot (1 - |\vec{g} \cdot \vec{r}|) $$
    Where $\|\vec{g}\|$ is the magnitude of the gradient (represented by `n.a` in the source code, assuming `n.rgb` is normalized or the magnitude is stored in `n.a`), and $\vec{r}$ is the unit ray direction vector [S1]. Note: The source code uses `n.a` as the magnitude multiplier and `n.rgb` as the vector for the dot product, implying `n.a` represents $\|\vec{g}\|$ or a scalar weight derived from it [S1].
6.  **Compute Final Alpha:** Apply the `smoothstep` function to the factor $F$ using `stepMin` and `stepMax`:
    $$ \alpha_{final} = \text{smoothstep}(\text{stepMin}, \text{stepMax}, F) $$
7.  **Set Output Color:** Replace the sample's color with a solid red color (`vec4(1.0, 0.0, 0.0, alpha)`) and set its alpha to $\alpha_{final}$ [S1]. This creates a high-contrast edge highlight.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def edge_enhancement_shader(volume_index, pos, ray_dir, sample_volume_with_tf, compute_gradient):
    # 1. Sample volume with transfer function
    sample = sample_volume_with_tf(volume_index, pos)
    
    # 2. Check sample alpha threshold
    if sample.a > 0.1:
        # 3. Compute gradient
        # Returns vec4: rgb = gradient vector g, a = gradient magnitude/mask
        n = compute_gradient(volume_index, pos, 0.0005)
        
        # 4. Check gradient validity
        if n.a > 0.0:
            # 5. Calculate factor: ||g|| * (1 - |g . r|)
            # Note: n.rgb is g, n.a is ||g|| (or magnitude proxy)
            gradient_vector = n.rgb
            gradient_magnitude = n.a
            ray_direction = ray_dir
            
            # Dot product of gradient and ray direction
            dot_product = dot(gradient_vector, ray_direction)
            
            # Factor calculation
            factor = gradient_magnitude * (1.0 - abs(dot_product))
            
            # 6. Compute final alpha using smoothstep
            step_min = 0.02
            step_max = 0.06
            alpha_final = smoothstep(step_min, step_max, factor)
            
            # 7. Set output to red edge highlight
            return vec4(1.0, 0.0, 0.0, alpha_final)
    
    # If conditions not met, return original sample
    return sample
```

**Edge Cases and Pitfalls:**
*   **Zero Gradient:** If the gradient magnitude is zero (homogeneous region), the factor is zero, and the alpha becomes zero (or very low depending on smoothstep), effectively hiding the region [S1].
*   **Parallel Gradient/Ray:** If the gradient is parallel to the ray direction ($\vec{g} \cdot \vec{r} \approx \pm 1$), the term $(1 - |\vec{g} \cdot \vec{r}|)$ approaches zero, reducing edge enhancement. This is intentional, as edges perpendicular to the view are the primary depth cue [S1].
*   **Precision:** The use of `smoothstep` requires careful tuning of `stepMin` and `stepMax` to avoid binary alpha results or overly soft edges [S1].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample, as it involves a constant number of vector operations and a smoothstep evaluation.
*   **Memory Complexity:** $O(1)$ additional memory per sample for temporary variables.
*   **Performance:** The method adds minimal computational overhead to the volume rendering pipeline, primarily requiring the gradient computation step which is already part of the preprocessing or shader stage [S1].
*   **Strengths:** Significantly improves depth perception for complex, overlapping structures like vasculature by providing strong ordinal depth cues [S1].
*   **Limitations:** The method produces a specific red edge highlight, which may not be suitable for all visualization contexts without color mapping adjustments. It relies heavily on the accuracy of the gradient computation [S1].
*   **When to Use:** Ideal for medical visualization (e.g., CTA scans) where disambiguating overlapping vessels is critical [S1].
*   **When Not to Use:** When naturalistic color rendering is required without artificial edge highlighting, or when performance is extremely constrained and gradient computation is too expensive.

## Background
The technique was introduced in the context of the PRISM framework, an open-source tool for interactive GPU volume rendering shader design [S1]. It is presented alongside other depth perception enhancement methods such as Chroma-depth and aerial perspective [S1]. The specific implementation shown in the source uses a red color for edge highlights, which can be combined with standard Blinn-Phong shading for illustrative purposes [S1]. The method builds upon the Voxel Gradient Calculation step, which provides the necessary gradient data [S1].

## Sources
[S1] PRISM: An open source framework for the interactive design of GPU volume rendering shaders., https://doi.org/10.1371/journal.pone.0193636, fetched 2026-10-26
