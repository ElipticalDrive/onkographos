# Adaptive Minimum Projected Size Adjustment

Also called: adaptive LOD control, framerate-driven projected size scaling

## Overview
Adaptive Minimum Projected Size Adjustment is a dynamic level-of-detail (LOD) control mechanism for isosurface extraction that automatically adjusts the minimum projected size of surface elements frame-to-frame to maintain a target framerate during user interaction [S1]. Unlike fixed minimum projected size approaches that cause surface complexity to scale directly with camera distance, this method balances geometric detail and rendering performance based on actual frame timing, preventing poor or over-populated geometry at varying camera distances [S1].

## How it works
**Inputs and Outputs**
*   **Input:** The current frame's elapsed rendering time, the desired display time (target frame duration), and the current minimum projected size parameter used by the isosurface extraction algorithm.
*   **Output:** An updated minimum projected size value for the next frame's extraction pass.

**Data Structures**
No specific complex data structures are required for the adjustment logic itself; it operates on scalar values representing time and size. The minimum projected size is a scalar parameter passed to the underlying resolution-adaptive volume sculpting system [S1].

**Steps**
1.  **Measure Frame Time:** At the end of each rendering frame, measure the elapsed time ($t_{measured}$) since the end of the previous frame [S1].
2.  **Calculate Time Difference:** Compute the difference ($\Delta t$) between the measured frame time and the desired display time ($t_{target}$):
    $$ \Delta t = t_{measured} - t_{target} $$
3.  **Compute Weighted Growing Factor:** Raise the time difference to the third power to compute the adjustment factor ($f$), preserving the sign of the difference:
    $$ f = (\Delta t)^3 $$
    This cubic weighting minimizes the influence of the adjustment when the display time is near the goal and emphasizes it when the display time deviates significantly, ensuring smooth convergence and preventing oscillation [S1].
4.  **Update Minimum Projected Size:** Update the minimum projected size ($S_{new}$) using the computed weighted growing factor. The exact accumulation formula is not explicitly detailed in the source, but the factor $f$ is applied to grow or shrink the size limit [S1].
5.  **Idle Reduction:** When the user is idle (no interaction), progressively reduce the minimum projected size toward zero. This allows fully detailed geometry to be rendered over time, enabling an accurate non-interactive display during the session [S1].

**Parameters**
*   **$t_{target}$ (desired_display_time):** Target frame duration used as the baseline for time-difference calculation to maintain interaction framerate. Value: Not given in the sources.
*   **Exponent:** The exponent applied to the time difference. Value: 3 [S1].

**Pitfalls and Edge Cases**
*   **Oscillation:** If the display time deviates significantly from the goal, the cubic weighting emphasizes the adjustment. If close, it minimizes influence. This design choice prevents oscillation and ensures smooth convergence [S1].
*   **Cracks:** The method deliberately ignores cracks that appear between adjacent cells of different sizes in multiresolution isosurface constructions. Solving this by drawing more triangles is not considered a solution because the problem may stem from the field not being defined at the same resolution in adjacent cells. The method does not reconnect surface elements found at each cell [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_minimum_projected_size(current_size, measured_time, target_time, is_user_idle):
    """
    Updates the minimum projected size for isosurface extraction based on frame timing.
    
    Args:
        current_size (float): The minimum projected size from the previous frame.
        measured_time (float): The time taken to render the current frame.
        target_time (float): The desired display time (target frame duration).
        is_user_idle (bool): True if the user is not interacting with the scene.
        
    Returns:
        float: The updated minimum projected size for the next frame.
    """
    # Step 1 & 2: Calculate time difference
    delta_t = measured_time - target_time
    
    # Step 3: Compute weighted growing factor (cubic weighting)
    # Preserves sign of delta_t
    factor = delta_t ** 3
    
    # Step 4: Update size
    # Note: The exact accumulation formula (e.g., additive vs multiplicative) 
    # is not specified in the source. We assume an additive update based on 
    # "weight the growing factor".
    # [G] Guard against negative size if factor is large and negative
    new_size = current_size + factor
    if new_size < 0:
        new_size = 0.0
        
    # Step 5: Idle reduction
    if is_user_idle:
        # Progressively reduce size toward zero
        # [G] Linear decay towards zero is a standard interpretation of "progressively reduce"
        # when no specific decay rate is given.
        new_size = new_size * 0.9 # Example decay factor, exact rate not given
        if new_size < 1e-6:
            new_size = 0.0
            
    return new_size
```

## Subtopics

### Adaptive LOD Control
*   **What it is:** A broader category of techniques that adjust the level of detail based on system performance or user interaction.
*   **Difference:** Adaptive Minimum Projected Size Adjustment is a specific implementation within this category that uses frame time feedback to drive the minimum projected size parameter of an isosurface extractor.
*   **When to choose:** Use when interactive performance is critical and the scene complexity varies with camera distance or user action.
*   **Link:** [Adaptive Minimum Projected Size Adjustment](8340-adaptive-minimum-projected-size-adjustment.md)

### Framerate-Driven Projected Size Scaling
*   **What it is:** The specific mechanism of scaling geometric parameters based on measured framerate.
*   **Difference:** This is the core mechanism of the method, distinguishing it from distance-based or heuristic-based LOD systems.
*   **When to choose:** Use when maintaining a stable framerate is more important than maintaining a consistent geometric resolution.
*   **Link:** [Adaptive Minimum Projected Size Adjustment](8340-adaptive-minimum-projected-size-adjustment.md)

## Operating characteristics
*   **Time Complexity:** The adjustment logic itself is $O(1)$, involving basic arithmetic operations. The overall complexity depends on the underlying isosurface extraction algorithm, which is typically linear with respect to the number of voxels/cells processed.
*   **Memory Complexity:** $O(1)$ for the adjustment state. The memory usage is dominated by the volume data and the generated mesh.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** The method scales with the frame rate. As the frame time approaches the target, adjustments become smaller, leading to stable behavior.
*   **Strengths:** Maintains interactive framerate during user interaction; allows high detail when idle; simple to implement.
*   **Limitations:** Does not address cracks between adjacent cells of different resolutions; relies on accurate frame time measurement; may cause visible popping if the transition between LOD levels is not smooth.
*   **When to use:** Interactive volume sculpting and visualization where user interaction speed is variable.
*   **When not to use:** Offline rendering where consistent quality is required; scenes where crack-free transitions between LOD levels are mandatory without post-processing.

## Background
The method was introduced by Eric Ferley, Marie-Paule Cani, and Jean-Dominique Gascuel in the context of "Resolution Adaptive Volume Sculpting" [S1]. It was presented at the 2001 Symposium on Geometry Processing (FCG01) [S1]. The work addresses the limitations of fixed minimum projected size controls in interactive modeling systems, where surface complexity could become poorly populated or over-populated depending on camera distance and user interaction [S1].

## Sources
[S1] Resolution Adaptive Volume Sculpting, https://doi.org/10.1006/gmod.2001.0558, fetched 2026-10-26
