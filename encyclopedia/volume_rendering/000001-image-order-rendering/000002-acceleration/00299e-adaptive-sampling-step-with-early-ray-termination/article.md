# Adaptive Sampling Step with Early Ray Termination

Also called: Pas d'échantillonnage volumique et arrêt précoce

## Overview
This method optimizes volume rendering for distant data by dynamically decreasing the sampling frequency based on the distance from the viewpoint, combined with early ray termination to skip fully opaque segments [S1, unofficial]. It sets itself apart from uniform sampling by addressing the visual degradation and computational waste associated with distant, low-detail regions, while distinguishing itself from sibling methods like [Ray-Centric Adaptive Step and Termination](5938-ray-centric-adaptive-step-and-termination.md) by specifically targeting brick-based procedural volumes and integrating a 3D pre-integrated transfer function to maintain brightness consistency across varying step sizes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin and direction, volume data represented as bricks, a 3D pre-integrated transfer function texture, and a threshold for total opacity.
*   **Output:** Final pixel color and opacity.

**Data Structures:**
*   **Bricks:** Spatial partitions of the volume. Each brick stores optical properties, potentially including a mean value for constant bricks to allow single-step accumulation [S1, unofficial].
*   **3D Pre-integrated Transfer Function:** A 3D texture storing pre-computed optical properties (color and opacity) for specific step sizes and input values, allowing consistent brightness regardless of sampling frequency [S1, unofficial].

**Steps:**
1.  **Ray Casting:** Cast a ray through the volume.
2.  **Distance-Based Step Adaptation:** For each sample point within a brick, calculate the distance $d$ from the viewpoint. Adapt the sampling step size $\Delta t$ for subsequent samples in that brick based on $d$. Distant samples use larger steps (lower frequency), while close samples use smaller steps (higher frequency) [S1, unofficial].
3.  **Pre-integrated Lookup:** Instead of using a standard transfer function that assumes a constant step size, use the 3D pre-integrated transfer function. This function maps the input scalar value and the current adaptive step size $\Delta t$ to the accumulated color and opacity contribution for that segment [S1, unofficial]. This ensures that the change in step size does not cause inconsistencies in opacity accumulation or brightness [S1, unofficial].
4.  **Brick Accumulation:** Accumulate the optical properties (color and opacity) for the entire brick in a single step if the brick is constant, or by summing the contributions of samples within the brick using the pre-integrated function [S1, unofficial].
5.  **Early Ray Termination Check:** After accumulating the contribution of each brick (not after every individual sample), check if the accumulated opacity $\alpha_{accum}$ has reached or exceeded a predefined threshold (e.g., 1.0 or a value close to it, such as 0.99) [S1, unofficial].
6.  **Termination:** If the opacity threshold is reached, terminate the ray evaluation early and return the accumulated color. Otherwise, continue to the next brick [S1, unofficial].

**Pitfalls:**
*   **Opacity Inconsistency:** Using a standard transfer function with an adaptive step size leads to incorrect opacity accumulation and brightness variations because the transfer function assumes a fixed integration interval [S1, unofficial]. This is mitigated by using the 3D pre-integrated transfer function.
*   **Performance Overhead of Checks:** Performing the early termination check after every sample can negatively impact performance due to dynamic branching, especially on older GPU architectures like the G80 [S1, unofficial]. Therefore, the check is performed after each brick accumulation.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray(ray_origin, ray_direction, bricks, pre_integrated_tf, opacity_threshold):
    accumulated_color = (0.0, 0.0, 0.0)
    accumulated_opacity = 0.0
    background_color = (0.0, 0.0, 0.0) # Assume black background

    for brick in bricks:
        # Calculate distance from viewpoint to the current brick or sample
        # This determines the adaptive step size
        distance = calculate_distance(ray_origin, brick)
        
        # Adapt step size based on distance
        # Distant bricks use larger steps
        step_size = adapt_step_size(distance)
        
        # Accumulate optical properties for the brick
        # Use pre-integrated transfer function to handle varying step sizes
        brick_color, brick_opacity = accumulate_brick(
            ray_origin, 
            ray_direction, 
            brick, 
            pre_integrated_tf, 
            step_size
        )
        
        # Composite the brick's contribution
        # Standard volume rendering equation: C_new = C_old + C_brick * alpha_brick * (1 - alpha_old)
        # O_new = O_old + alpha_brick * (1 - O_old)
        
        # Note: accumulate_brick returns the contribution of the brick segment
        # The exact compositing depends on the specific implementation of accumulate_brick
        # Assuming it returns the color and opacity contribution for the segment
        
        # Composite
        alpha_contribution = brick_opacity * (1.0 - accumulated_opacity)
        accumulated_color = accumulated_color + brick_color * alpha_contribution
        accumulated_opacity = accumulated_opacity + alpha_contribution
        
        # Early Ray Termination Check: After each brick
        if accumulated_opacity >= opacity_threshold:
            break
            
    # If ray didn't terminate early, add background
    if accumulated_opacity < 1.0:
        accumulated_color = accumulated_color + background_color * (1.0 - accumulated_opacity)
        
    return accumulated_color
```

## Subtopics

### [Adaptive Sampling with Monotonically Decreasing Density Function](5388-adaptive-sampling-with-monotonically-decreasing-density-func.md)
This method determines sampling density along a ray using a monotonically decreasing function of distance from the eye, simulating photographic clarity where distant objects are vague (sparse sampling) and close objects are clear (dense sampling). It improves upon the Weghorst et al. method by stabilizing the step calculation, preventing the under-sampling and aliasing artifacts that occur when using simple inverse distance functions. It achieves higher structural similarity (SSIM > 0.95) with significantly reduced execution time compared to uniform linear sampling.

## Operating characteristics
*   **Performance:** Provides good performance on GPU architectures like the G80, particularly when early exit conditions are effective [S1, unofficial].
*   **Quality:** Maintains constant brightness and provides better rendering quality compared to uniform sampling with adaptive steps, especially when using the 3D pre-integrated transfer function [S1, unofficial].
*   **Limitations:** The effectiveness of early termination depends on the opacity distribution in the volume. If opacity builds up slowly, the benefit is reduced. The method is specifically designed for brick-based procedural volumes.
*   **Complexity:** The use of a 3D pre-integrated transfer function adds memory overhead for the texture but simplifies the per-sample computation by handling the integration over the adaptive step size.

## Background
This technique originates from research on interactive exploration of extended and detailed procedural volumes [S1, unofficial]. It was developed to address the computational cost of rendering large, distant datasets where high-frequency sampling is unnecessary. The integration of early ray termination and adaptive step sizes was optimized for GPU execution, specifically targeting architectures like the NVIDIA G80 [S1, unofficial]. The use of 3D pre-integrated transfer functions was introduced to solve the inconsistency issues arising from variable step sizes in volume rendering.

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
