# Adaptive Sampling Step with Early Ray Termination

Also called: Pas d'échantillonnage volumique et arrêt précoce d'un rayon

## Overview
This method optimizes the rendering of distant volumetric data by dynamically decreasing the sampling frequency based on the distance from the viewpoint, combined with early ray termination to skip unnecessary computation once opacity is saturated [S1, unofficial]. It distinguishes itself from uniform sampling by adapting the step size to spatial coherence, requiring a pre-integrated transfer function to maintain brightness consistency across varying step sizes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin and direction; 3D pre-integrated transfer function texture; volumetric data organized in bricks.
*   **Output:** Final pixel color and opacity.

**Data Structures:**
*   **3D Pre-integrated Transfer Function:** A 3D texture storing optical properties (color and opacity) integrated over a range of step sizes. Accessed via floating-point coordinates representing the material properties and the effective step size [S1, unofficial].
*   **Bricks:** Volumetric data chunks. Each brick stores accumulated properties, potentially including a mean value for constant regions to allow single-step accumulation [S1, unofficial].

**Algorithm Steps:**
1.  **Initialize:** Set accumulated opacity $\alpha_{acc} = 0$ and accumulated color $C_{acc} = 0$.
2.  **Ray Marching Loop:** For each brick intersected by the ray:
    a.  **Calculate Adaptive Step:** Determine the sampling step size $\Delta s$ based on the distance of the current sample position to the viewpoint. Distant samples use larger steps; close samples use smaller steps [S1, unofficial].
    b.  **Sample Volume:** Evaluate the volume density and material properties at the current position using the calculated step size.
    c.  **Lookup Pre-integrated Properties:** Use the 3D pre-integrated transfer function to retrieve the color contribution and opacity increment corresponding to the current material properties and the adaptive step size $\Delta s$. This ensures brightness consistency despite varying $\Delta s$ [S1, unofficial].
    d.  **Accumulate:** Update accumulated opacity and color using volume rendering equation:
        $$ \alpha_{new} = \alpha_{acc} + (1 - \alpha_{acc}) \cdot \alpha_{step} $$
        $$ C_{new} = C_{acc} + (1 - \alpha_{acc}) \cdot \alpha_{step} \cdot C_{step} $$
        where $\alpha_{step}$ and $C_{step}$ are retrieved from the pre-integrated transfer function.
    e.  **Early Termination Check:** After accumulating each brick (not every sample), check if $\alpha_{acc}$ has reached a threshold for total opacity (e.g., $1.0$ or a value very close to it). If so, terminate the ray evaluation early and return the current accumulated color [S1, unofficial].
    f.  **Update Position:** Advance the ray position by the adaptive step size $\Delta s$.
3.  **Return:** If the loop completes without early termination, return $C_{new}$.

**Parameters:**
*   **Opacity Threshold:** The value at which early termination is triggered. Not given in the sources.
*   **Step Size Function:** The specific function mapping distance to step size. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Opacity Inconsistency:** Using an adaptive step size with a standard transfer function (assuming constant step size) causes inconsistency in opacity accumulation and brightness [S1, unofficial]. This is mitigated by using the 3D pre-integrated transfer function [S1, unofficial].
*   **Performance Impact of Dynamic Tests:** Early termination checks must be performed after brick accumulation, not after every sample, to avoid the performance penalty of dynamic tests in shaders [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray(ray_origin, ray_direction, bricks, pre_integrated_tf):
    alpha_acc = 0.0
    color_acc = 0.0
    
    for brick in bricks:
        if not ray_intersects_brick(ray_origin, ray_direction, brick):
            continue
            
        # Calculate adaptive step size based on distance to viewpoint
        # Distance calculation depends on specific implementation details not provided
        step_size = calculate_adaptive_step(ray_origin, brick.center)
        
        # Sample volume and get pre-integrated properties
        # Material properties and step_size are used to index the 3D texture
        alpha_step, color_step = pre_integrated_tf.lookup(material_properties, step_size)
        
        # Accumulate
        alpha_acc = alpha_acc + (1.0 - alpha_acc) * alpha_step
        color_acc = color_acc + (1.0 - alpha_acc) * alpha_step * color_step
        
        # Early termination check after each brick
        if alpha_acc >= 0.99: # Threshold not specified in sources
            break
            
        # Advance ray
        ray_origin = ray_origin + ray_direction * step_size
        
    return color_acc
```

## Subtopics

### [Adaptive Sampling with Monotonically Decreasing Density Function](5388-adaptive-sampling-with-monotonically-decreasing-density-func.md)
Determines sampling density using a monotonically decreasing function of distance, stabilizing step calculation and preventing aliasing artifacts common with simple inverse distance functions. Choose when higher structural similarity (SSIM > 0.95) and reduced execution time compared to uniform sampling are critical.

## Operating characteristics
*   **Time Complexity:** Reduced for distant data due to fewer samples per unit distance. Early termination further reduces computation for opaque regions.
*   **Memory Complexity:** Requires storage for the 3D pre-integrated transfer function texture.
*   **Measured Performance:** Provides good performance on the G80 GPU [S1, unofficial].
*   **Strengths:** Optimizes rendering of distant data; maintains brightness consistency with pre-integration; early termination saves computation.
*   **Limitations:** Requires pre-integrated transfer function; early termination check placement is critical for performance.
*   **When to Use:** Rendering large, distant volumetric datasets where performance is a concern and brightness consistency is important.
*   **When Not to Use:** When uniform sampling is sufficient or when pre-integration is not feasible.

## Background
Originated in research on interactive exploration of extended and detailed procedural volumes [S1, unofficial]. The technique was developed to optimize rendering on GPUs like the G80, addressing the need for efficient handling of distant data and early termination of opaque rays [S1, unofficial].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
