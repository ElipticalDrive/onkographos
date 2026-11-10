# Incremental Bubble-Sort A-Buffer Blending

Also called: A-Buffer algorithm, incremental bubble-sort

## Overview
Incremental Bubble-Sort A-Buffer Blending is a hardware-accelerated compositing technique for stroke-based splatting that simulates tint blending by applying alpha blending to rendered patches with stronger influence for topmost fragments [S1]. It differs from standard A-Buffer implementations by dividing the sorting and blending execution across multiple frames to maintain interactivity, whereas standard A-Buffer suffers from high complexity that compromises real-time performance [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A stream of fragments (brush strokes) generated from point cloud data, each defined by position, Gaussian parameters ($\sigma_x, \sigma_y$), amplitude, and color.
*   **Output:** A final composited image where high-frequency noise is removed and boundary regions between colors are blended smoothly [S1].

**Data Structures:**
The method utilizes texture memory on the graphics card to implement the A-Buffer [S1]. The A-Buffer stores fragments sorted by depth. While the exact bit-packing of the A-Buffer entries is not specified, the structure must support storing fragment color, alpha, and depth information to facilitate the bubble-sort operations [S1].

**Parameters:**
*   **Amplitude, $\sigma_x$, $\sigma_y$:** Values of the Gaussians that control the transparency component of the generated brush strokes [S1].
*   **Preset Iterations:** A threshold value determining the minimum number of bubble-sort iterations performed before the sorted data is used for final blending [S1].

**Algorithm Steps:**
1.  **Initialization:** Prepare the A-Buffer in texture memory. Initialize the system to perform Gaussian blur blending with Z-buffer occlusion as a fallback or interim step [S1].
2.  **Incremental Sorting:** Divide the execution of the bubble-sort algorithm across frames [S1].
3.  **Per-Frame Iteration:**
    a.  Perform one iteration of the bubble-sort algorithm. This specific choice of sorting technique ensures that each iteration orders exactly one new fragment into its correct position within the fragment array [S1].
    b.  Identify the newly sorted fragment.
    c.  Calculate the alpha-blended color using this new fragment and the previously accumulated color.
    d.  Store the result in the last position of the fragment array [S1].
4.  **Simplification:** By storing the blended result in the last position, subsequent frames simplify the color calculation operation to a simple mixing of two colors [S1].
5.  **Interim Rendering:** While the sorting is not yet complete, perform Gaussian blur blending with Z-buffer occlusion to provide a visually acceptable result [S1].
6.  **Final Blending:** Only use the fully sorted data after a preset amount of iterations has elapsed. This ensures a minimum amount of fragments are available to calculate a blended color of neighboring patches [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global state maintained in texture memory
# a_buffer: Array of fragments sorted by depth
# current_blended_color: Accumulated color from previous frames/iterations
# iteration_count: Number of bubble-sort iterations performed so far
# preset_iterations: Threshold for using sorted data

def process_frame(fragments):
    """
    Process a single frame of stroke-based splatting.
    """
    # Step 1: Interim rendering using Z-buffer and Gaussian blur
    # This ensures visual continuity while sorting is incomplete [S1]
    render_with_zbuffer_and_gaussian_blur(fragments)

    # Step 2: Incremental Bubble-Sort Iteration
    # Each iteration orders exactly one new fragment [S1]
    bubble_sort_iteration(a_buffer)

    # Step 3: Update Blended Color
    # Use the newly sorted fragment to calculate alpha-blended color [S1]
    # Store result in the last position of the fragment array [S1]
    new_fragment = get_last_sorted_fragment(a_buffer)
    current_blended_color = alpha_blend(current_blended_color, new_fragment)

    # Step 4: Check if sorted data is ready for final output
    iteration_count += 1
    
    if iteration_count >= preset_iterations:
        # Use the sorted data to calculate blended color of neighboring patches [S1]
        final_image = render_from_sorted_a_buffer(a_buffer)
    else:
        # Continue using interim rendering or partial blend
        final_image = current_blended_color

    return final_image

def bubble_sort_iteration(a_buffer):
    """
    Perform one pass of bubble sort to order one fragment.
    Standard bubble sort logic applied to the A-Buffer array.
    """
    n = length(a_buffer)
    for i from 0 to n - 2:
        if a_buffer[i].depth > a_buffer[i+1].depth:
            swap(a_buffer[i], a_buffer[i+1])
```

**Edge Cases and Pitfalls:**
*   **High-Frequency Noise:** High-frequency noise is removed even with high amplitude values and low sigma values (opaque brush) because the falloff of the Gaussian function is not instantaneous due to its continuity [S1].
*   **Interactivity vs. Complexity:** Standard A-Buffer has high complexity which compromises interactivity. The incremental approach mitigates this but requires careful tuning of the `preset_iterations` to balance quality and speed [S1].

## Operating characteristics
*   **Time Complexity:** The standard A-Buffer algorithm has high complexity [S1]. The incremental bubble-sort approach reduces the per-frame cost by dividing the sorting work across frames, with each frame performing only one iteration of bubble sort, which orders one new fragment [S1].
*   **Memory Complexity:** Relies on texture memory on the graphics card to implement the A-Buffer [S1]. The exact memory footprint depends on the number of fragments and the resolution of the A-Buffer, which is not specified in the sources.
*   **Performance:** Designed to maintain interactivity during camera navigation or when applied to moving objects [S1].
*   **Strengths:** Simulates tint blending closely; removes high-frequency noise effectively; maintains interactivity through incremental processing [S1].
*   **Limitations:** Requires a preset number of iterations before final sorted data is used, potentially causing visual artifacts during rapid changes if the threshold is not met [S1].

## Background
This technique was developed as an alternative blending method for stroke-based splatting, a multi-resolution point cloud visualization technique [S1]. It was implemented to improve visualization quality, particularly for boundary regions between colors, by simulating the process of tint blending [S1]. The method was introduced in the paper "Stroke-based splatting: an efficient multi-resolution point cloud visualization technique" [S1].

## Sources
[S1] Stroke-based splatting: an efficient multi-resolution point cloud visualization technique, https://doi.org/10.1007/s00371-017-1420-7, fetched 2026-10-26
