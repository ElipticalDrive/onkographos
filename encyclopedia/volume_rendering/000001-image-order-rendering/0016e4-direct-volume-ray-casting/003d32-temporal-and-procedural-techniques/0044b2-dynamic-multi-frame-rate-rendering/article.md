# Dynamic Multi-Frame Rate Rendering

Also called: Dynamic single and multi-frame rate rendering

## Overview
Dynamic Multi-Frame Rate Rendering is a hybrid rendering system that dynamically switches between multi-frame rate rendering (using image-based warping for low latency) and single-frame rate parallel raycasting (using all GPUs for high quality) based on current workload complexity [S1]. It solves the trade-off between the low latency of multi-frame rate techniques and the high quality/artifact-free nature of parallel raycasting by selecting the optimal method for the current frame [S1]. It sets itself apart from its siblings by explicitly managing the transition between these two distinct rendering paradigms based on a runtime decision function, whereas siblings like [Volumetric Smoke Rendering with Worley Noise and Light Maps](2447-volumetric-smoke-rendering-with-worley-noise-and-light-maps.md) focus on procedural density and [GPU-accelerated ray casting with volumetric lenses](6767-gpu-accelerated-ray-casting-with-volumetric-lenses.md) focus on specific optical effects [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current rendering parameters including sampling rate, zoom level (resolution), occlusion, dataset size, and transfer function [S1].
*   **Outputs:** A selected rendering mode (Multi-Frame Rate or Single-Frame Rate Parallel Raycasting) and the corresponding rendered image [S1].

**Data Structures:**
*   **Decision Function:** A lookup table or mathematical model mapping input parameters (sampling rate, resolution) to the superior rendering method [S1]. The structure is not explicitly defined in the sources, but it is evaluated periodically [S1].
*   **Performance Data:** Pre-computed or approximated performance metrics for different parameter combinations [S1].

**Numbered Steps:**
1.  **Collect Performance Data:** Gather performance data for the relevant parameters (sampling rate, zoom level, occlusion, dataset size, transfer function) to form a decision function that yields the best rendering method for the current frame [S1]. This measurement takes approximately 2 minutes for full evaluation, but a good approximation can be obtained within 10 seconds [S1].
2.  **Evaluate Decision Function:** Periodically evaluate the decision function during runtime using the current values of the performance-relevant parameters [S1].
3.  **Select Rendering Method:**
    *   If the volume rendering complexity exceeds a break-even point (e.g., high sampling rate, high resolution), use **Multi-Frame Rate Rendering** [S1]. This method uses image-based warping for low latency [S1].
    *   If the complexity is below the break-even point (e.g., low sampling rate, low resolution), converge to **Single-Frame Rate Multi-GPU Raycasting** (specifically sort-first parallel raycasting) [S1]. This utilizes all available GPUs and avoids image warping artifacts [S1].
4.  **Render:** Execute the selected rendering method for the current frame [S1].
5.  **Re-evaluate:** If performance-relevant parameters not input to the function change, recompute the performance data for the decision function [S1].

**Formulas:**
*   **Resolution Measurement:** Resolution is measured by computing the pixel area of the bounding rectangle, serving as a measurement of the model's zoom level [S1].
    $$ \text{Resolution} = \text{Pixel Area of Bounding Rectangle} $$
*   **Sampling Rate Range:** Sampling rate ranges from 6x undersampling to 6x oversampling compared to Nyquist rate sampling [S1].

**Parameters:**
*   **Sampling Rate:** A parameter affecting raycasting complexity, ranging from 6x undersampling to 6x oversampling compared to Nyquist rate sampling [S1].
*   **Resolution:** Measured by computing the pixel area of the bounding rectangle [S1].

**Edge Cases and Pitfalls:**
*   **Convergence to Single-Frame Rate:** When converging to single-frame rate rendering (e.g., $m=0$ in quality map approach), additional GPUs become useless as raycasting is performed entirely by the fast node [S1].
*   **Parameter Changes:** Performance data for the decision function must be recomputed if performance-relevant parameters not input to the function change [S1].
*   **Break-Even Point:** The system must accurately identify the break-even point where multi-frame rate rendering becomes beneficial over parallel raycasting [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Dynamic Multi-Frame Rate Rendering
# Assumes a pre-computed or approximated decision function `get_best_method`
# which takes sampling_rate and resolution as inputs and returns 'MFR' or 'PARALLEL'

def render_frame(sampling_rate, resolution, dataset, transfer_function, occlusion):
    """
    Renders a single frame using the optimal method based on complexity.
    
    Args:
        sampling_rate (float): Sampling rate relative to Nyquist (e.g., 0.5 to 6.0).
        resolution (int): Pixel area of the bounding rectangle.
        dataset (VolumeDataset): The volumetric dataset.
        transfer_function (TF): The transfer function for opacity/color.
        occlusion (float): Occlusion factor.
        
    Returns:
        Image: The rendered frame.
    """
    # Step 1: Evaluate the decision function
    # The decision function maps parameters to the best rendering method.
    # It is evaluated periodically, but for this frame, we use current values.
    best_method = get_best_method(sampling_rate, resolution)
    
    # Step 2: Select and execute the rendering method
    if best_method == 'MFR':
        # Multi-Frame Rate Rendering: Uses image-based warping for low latency.
        # The fast node achieves low latency, slow nodes act as frame source.
        # The fast node is not synchronized with slow nodes.
        image = render_multi_frame_rate(dataset, transfer_function, occlusion)
    elif best_method == 'PARALLEL':
        # Single-Frame Rate Parallel Raycasting: Uses all GPUs for high quality.
        # Avoids image warping artifacts.
        # Note: If complexity is very low, additional GPUs may be underutilized.
        image = render_parallel_ray_casting(dataset, transfer_function, occlusion)
    else:
        raise ValueError("Unknown rendering method selected by decision function.")
        
    return image

def get_best_method(sampling_rate, resolution):
    """
    Returns the best rendering method based on the decision function.
    The decision function is derived from performance data collected over time.
    Blue region in decision function (low complexity) -> PARALLEL
    Other regions (high complexity) -> MFR
    """
    # This function encapsulates the decision logic.
    # In practice, this might be a lookup table or a simple threshold check.
    # For example, if sampling_rate * resolution < threshold, use PARALLEL.
    # The sources indicate that sort-first parallel raycasting is superior in the "blue region" (low complexity).
    
    # Placeholder for the actual decision logic derived from performance data.
    # The sources do not provide the exact formula for the decision function,
    # only that it is based on sampling rate and resolution.
    # We assume a threshold-based approach for simplicity, as the exact function is not given.
    # A good approximation of the decision function can be obtained within 10 seconds.
    
    # Example threshold (not from sources, just for structure):
    # threshold = 1000 # Arbitrary value
    # if sampling_rate * resolution < threshold:
    #     return 'PARALLEL'
    # else:
    #     return 'MFR'
    
    # Since the exact decision function is not provided, we return a placeholder.
    # In a real implementation, this would be a lookup or a more complex function.
    return 'MFR' # Placeholder

def render_multi_frame_rate(dataset, transfer_function, occlusion):
    """
    Renders using Multi-Frame Rate technique.
    """
    # Implementation details of MFR are not provided in the sources.
    # It involves image-based warping and unbalanced workloads between fast and slow nodes.
    pass

def render_parallel_ray_casting(dataset, transfer_function, occlusion):
    """
    Renders using Single-Frame Rate Parallel Raycasting (sort-first).
    """
    # Implementation details of parallel raycasting are not provided in the sources.
    # It utilizes all available GPUs.
    pass
```

## Subtopics

### Multi-Frame Rate Rendering
Uses image-based warping for low latency. Superior when volume rendering complexity is high [S1]. Link: [Multi-Frame Rate Volume Rendering](https://doi.org/10.2312/egpgv.egpgv10.019-026).

### Single-Frame Rate Parallel Raycasting
Uses all GPUs for high quality, avoiding image warping artifacts. Superior when complexity is low [S1]. Link: [Direct volume ray casting](19-direct-volume-ray-casting.md).

## Operating characteristics
*   **Time Complexity:** The decision function evaluation has virtually no performance impact [S1]. Full performance data collection takes approximately 2 minutes, but a good approximation can be obtained within 10 seconds [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Tested on two datasets: dice (64^3 voxels) and MANIX (256x256x230 voxels) [S1]. Performance was measured for sampling rates from 6x undersampling to 6x oversampling compared to Nyquist rate sampling [S1]. The multi-frame rate system showed more stable frame rates than sort-first parallel raycasting in high complexity scenarios [S1].
*   **Scaling:** The system uses unbalanced workloads between a fast node (low latency) and slow nodes (frame source) [S1]. The fast node is not synchronized with slow nodes to increase speed [S1].
*   **Strengths:** Low latency (MFR), high quality and artifact-free (Parallel), adaptive to workload complexity [S1].
*   **Limitations:** When converging to single-frame rate rendering, additional GPUs become useless [S1]. Performance data must be recomputed if parameters change [S1].
*   **When to use:** Use when dynamic switching between low latency and high quality is required based on workload complexity [S1].
*   **When not to use:** Use when workload complexity is static and always low (parallel raycasting is sufficient) or always high (MFR is sufficient) [S1].

## Background
*   **Origin:** The method was described in the paper "Multi-Frame Rate Volume Rendering" by S. Hauswiesner, D. Kalkofen, and D. Schmalstieg [S1].
*   **History and Development:** The paper was published in 2010 by The Eurographics Association [S1]. It builds on the concept of multi-frame rate rendering and aims to address its limitations by dynamically switching to parallel raycasting when appropriate [S1].
*   **Where it is used:** Suitable for displaying transparent volumetric scenes at high frame rates and little latency to user input [S1].

## Sources
[S1] Multi-Frame Rate Volume Rendering, https://doi.org/10.2312/egpgv.egpgv10.019-026, fetched 2026-10-29
