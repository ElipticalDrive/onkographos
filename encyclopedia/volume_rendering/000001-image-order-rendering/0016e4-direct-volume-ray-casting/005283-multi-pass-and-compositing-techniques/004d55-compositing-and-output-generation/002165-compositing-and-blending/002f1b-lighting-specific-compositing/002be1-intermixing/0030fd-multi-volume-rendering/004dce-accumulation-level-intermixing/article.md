# Accumulation-level intermixing

Also called: Accumulation-level intermixing of two datasets, Multi-modal Rendering with Data Intermixing

## Overview
Accumulation-level intermixing is a multi-modal volume rendering technique that resolves the depth-ordering artifacts inherent in image-level intermixing by combining ray samples from multiple co-registered volumes at every step along the viewing ray [S1, unofficial]. Unlike image-level compositing, which blends final pixel colors and thus fails to preserve the correct occlusion relationships between distinct datasets, this method integrates physical properties (colors and opacities) from each volume's sample before the final accumulation, ensuring that the relative density and opacity of the constituent volumes are correctly composited [S1, unofficial]. It is distinguished from its sibling [Illumination model level intermixing](13581-illumination-model-level-intermixing.md) by applying separate lighting models to each volume's sample before combination, whereas illumination-level intermixing merges raw data values prior to any lighting calculation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Two or more co-registered volume datasets (e.g., CT and MRI) [S1, unofficial]. Each dataset provides scalar density values at sample points along a ray [G].
*   **Outputs:** A single composite color and opacity value for each pixel, correctly representing the occlusion and transparency relationships between the datasets [S1, unofficial].

**Data Structures:**
*   **Ray Sample:** For each volume $i$, a sample at position $t$ along the ray yields a color $C_i$ and an opacity $\alpha_i$ [S1, unofficial].
*   **Accumulator:** A front-to-back or back-to-front buffer storing the current accumulated color $C_{acc}$ and opacity $\alpha_{acc}$ [G].

**Steps:**
1.  **Ray Casting:** For each pixel, cast a ray through the volume space [G].
2.  **Sampling:** At each step $t$ along the ray, sample the density from each volume dataset $i$ [G].
3.  **Color and Opacity Assignment:** Compute the color $C_i(t)$ and opacity $\alpha_i(t)$ for each volume's sample independently [S1, unofficial]. In multi-modal scenarios (e.g., CT vs. MRI), colors can be assigned based on intensity comparison; for example, if CT intensity exceeds MRI intensity, the sample color is blue, and if MRI exceeds CT, it is green [S2, book]. For multi-channel data, gradients may be computed in $L^*u^*v^*$ space using the second derivative along the gradient direction to compute opacity [S2, book].
4.  **Mixing:** Combine the individual volume samples $(C_i, \alpha_i)$ according to a mixing function [S1, unofficial]. In inclusive opacity mode, the combined opacity $\alpha_{combined}$ for two volumes is calculated as $\alpha_{combined} = 1 - (1 - \alpha_{CT})(1 - \alpha_{MRI})$ [S1, unofficial]. Alternatively, samples can be mixed based on dominance, where the color of the volume with the higher intensity is chosen [S2, book].
5.  **Accumulation:** Composite the mixed sample $(C_{mixed}, \alpha_{mixed})$ into the ray accumulator using standard front-to-back compositing: $C_{acc} = C_{acc} + C_{mixed} \cdot \alpha_{mixed} \cdot (1 - \alpha_{acc})$ and $\alpha_{acc} = \alpha_{acc} + \alpha_{mixed} \cdot (1 - \alpha_{acc})$ [G].
6.  **Termination:** Repeat until the ray exits the volume or $\alpha_{acc}$ reaches 1.0 [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_accumulation_intermix(ray_origin, ray_dir, volumes, step_size):
    acc_color = [0.0, 0.0, 0.0]
    acc_alpha = 0.0
    t = 0.0
    while t < ray_length and acc_alpha < 1.0:
        # Sample all volumes at current step [G]
        samples = [vol.sample(ray_origin + ray_dir * t) for vol in volumes]
        # Compute color and opacity per volume [S1, unofficial]
        colors = [compute_color_opacity(s) for s in samples]
        # Mix according to function [S1, unofficial]
        mixed_color, mixed_alpha = mix_samples(colors)
        # Accumulate into ray buffer [G]
        acc_color = [c + mc * ma * (1.0 - aa) for c, mc, ma in zip(acc_color, mixed_color, mixed_alpha)]
        acc_alpha += mixed_alpha * (1.0 - acc_alpha)
        t += step_size
    return acc_color
```

**Parameters:**
*   **Mixing Function:** The specific formula for combining $(C_i, \alpha_i)$ varies [S1, unofficial].
*   **Intensity Thresholds:** For comparison-based coloring, thresholds or relative comparisons (e.g., $I_{CT} > I_{MRI}$) define the color mapping [S2, book].

**Edge Cases and Pitfalls:**
*   **Depth Ordering:** If image-level intermixing is used instead, depth ordering is lost [S1, unofficial]. Accumulation-level fixes this but requires careful handling of opacity addition to avoid over-saturation [G].
*   **Precision:** When using the inclusive opacity formula, if both $\alpha_{CT}$ and $\alpha_{MRI}$ are high, the combined opacity approaches 1 quickly, potentially obscuring deeper structures if not managed with proper transfer functions [G].
*   **Gradient Computation:** Computing gradients in color spaces like $L^*u^*v^*$ requires converting scalar densities to color values first [S2, book].

## Subtopics

### [Illumination model level intermixing](13581-illumination-model-level-intermixing.md)
Illumination model level intermixing combines ray samples from distinct volume datasets before computing final color and opacity values. It solves the problem of rendering heterogeneous data by integrating physical properties at the sample level, rather than blending pre-computed visual attributes. Unlike accumulation-level intermixing, which composites individual volume contributions after their respective lighting calculations, this method merges raw data to apply a unified lighting model. It is chosen when a single, physically consistent lighting model across all modalities is desired.

### [Acquisition Level Intermixing](17050-acquisition-level-intermixing.md)
Acquisition Level Intermixing fuses multiple spatially aligned scalar volumes into a single vectorial volume before the standard rendering pipeline begins. Unlike accumulation-level intermixing, which combines ray samples during traversal, this method performs statistical classification on the combined data to extract material properties, thereby preserving depth ordering and enabling joint analysis of dependent acquisitions. It is chosen when volumes represent different acquisitions of the same subject and statistical material classification is more valuable than direct visual intermixing [G].

### [Hardware-accelerated Multimodality Volume Fusion](17391-hardware-accelerated-multimodality-volume-fusion.md)
Hardware-accelerated Multimodality Volume Fusion combines multiple volume datasets into a single image representation using graphics hardware to solve depth cueing problems and reduce processing time compared to software-based methods. It distinguishes itself from sibling methods like [Illumination model level intermixing](13581-illumination-model-level-intermixing.md) by performing compositing operations on texture coordinates before alpha blending, thereby avoiding the need to manually set slice ordering for volumes of different sizes. It is chosen when real-time performance is critical and the hardware supports multi-texture blending operations [G].

## Operating characteristics
Time Complexity: Not given in the sources.
Memory Complexity: Not given in the sources.
Measured Performance: Not given in the sources.
Strengths: Correctly preserves depth ordering between different modalities; allows for complex visualizations where one modality occludes or modulates another; flexible mixing functions allow for various visual interpretations [S1, unofficial].
Limitations: Requires precise co-registration of volumes; computational cost increases linearly with the number of volumes; complex mixing functions may be harder to tune for perceptual clarity [G].
When to Use: Not given in the sources.
When Not to Use: Not given in the sources.

## Background
Accumulation-level intermixing was introduced as a solution to the limitations of image-level intermixing in multi-modal volume rendering [S1, unofficial]. Image-level intermixing, which blends the final rendered images of separate volumes, fails to preserve depth ordering, leading to visually incorrect results where one volume might incorrectly appear in front of another [S1, unofficial]. Accumulation-level intermixing addresses this by integrating samples at the ray-casting step [S1, unofficial]. The method is part of the broader family of multi-volume rendering techniques, which aim to visualize heterogeneous data sets together [S1, unofficial]. It was notably demonstrated in the context of the "Visible Man" dataset, combining CT and MRI data, and in radiation therapy dose planning [S1, unofficial]. The technique relies on standard front-to-back compositing but modifies the input to the compositing equation to be a mixture of multiple volume samples rather than a single volume sample [S2, book].

## Sources
[S1] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
