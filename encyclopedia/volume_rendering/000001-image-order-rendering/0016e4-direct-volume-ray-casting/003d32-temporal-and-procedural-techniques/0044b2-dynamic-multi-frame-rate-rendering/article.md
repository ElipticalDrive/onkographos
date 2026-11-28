# Dynamic Multi-Frame Rate Rendering

Also called: Dynamic single and multi-frame rate rendering

## Overview
Dynamic Multi-Frame Rate Rendering is a hybrid rendering system that dynamically switches between multi-frame rate rendering (using image-based warping for low latency) and single-frame rate parallel raycasting (using all GPUs for high quality) based on current workload complexity [S1]. It solves the trade-off between the low latency of fast-node rendering and the high throughput of parallel raycasting by selecting the optimal method for the current frame's specific parameters [S1]. Unlike its siblings, which focus on procedural evaluation or specific hardware acceleration, this method focuses on runtime adaptation of the rendering paradigm itself [S1].

## How it works
The method operates by periodically evaluating a decision function that predicts the performance of different rendering modes based on current scene parameters [S1].

**Inputs and Outputs:**
*   **Inputs:** Current rendering parameters including sampling rate, zoom level (resolution), occlusion, dataset size, and transfer function [S1].
*   **Outputs:** A categorical decision indicating whether to use Multi-Frame Rate (MFR) rendering or Single-Frame Rate (SFR) parallel raycasting [S1].

**Data Structures:**
*   **Decision Function:** A lookup table or mathematical model mapping input parameters (sampling rate, resolution) to predicted frame rates for both MFR and SFR modes, derived from empirical measurements [S1].

**Steps:**
1.  **Performance Data Collection:** Collect performance data for the relevant parameters (sampling rate, zoom level, occlusion, dataset size, transfer function) to form a decision function [S1]. This involves measuring the frame rate of both MFR and SFR modes across the parameter space [S1].
2.  **Decision Function Approximation:** The full evaluation of the decision function takes approximately 2 minutes, but a good approximation can be obtained within 10 seconds [S1].
3.  **Runtime Evaluation:** Periodically evaluate the decision function during runtime using the current values of the input parameters [S1].
4.  **Method Selection:** If the volume rendering complexity exceeds a break-even point (indicated by the decision function), use multi-frame rate rendering [S1]. Otherwise, converge to single-frame rate multi-GPU raycasting to utilize all available resources and avoid image warping artifacts [S1].
5.  **Re-evaluation Trigger:** Recompute the performance data for the decision function if any performance-relevant parameters not input to the function change [S1].

**Parameters:**
*   **Sampling Rate:** Ranges from 6x undersampling to 6x oversampling compared to the Nyquist rate sampling [S1].
*   **Resolution:** Measured by computing the pixel area of the bounding rectangle, serving as a measurement of the model's zoom level [S1].

**Pitfalls and Edge Cases:**
*   **Useless GPUs in SFR:** When converging to single-frame rate rendering (e.g., $m=0$ in the quality map approach), additional GPUs become useless as raycasting is performed entirely by the fast node [S1].
*   **Stale Decision Function:** Performance data for the decision function must be recomputed if performance-relevant parameters not input to the function change [S1].
*   **Precision:** The decision function evaluation has virtually no performance impact [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Precomputed decision function mapping (sampling_rate, resolution) -> (mfr_fps, sfr_fps)
# Derived from empirical measurements across the parameter space [S1]
decision_function = {}

def evaluate_decision_function(sampling_rate, resolution):
    """Queries the decision function for predicted frame rates [S1]."""
    return decision_function.get((sampling_rate, resolution), (0.0, 0.0))

def check_params_changed(other_params, last_params):
    """Checks if performance-relevant parameters not in the function key have changed [S1]."""
    return other_params != last_params

def recompute_decision_function(all_params):
    """Rebuilds the decision function from empirical measurements [S1]."""
    # Full evaluation takes ~2 minutes; a good approximation takes ~10 seconds [S1]
    # The exact measurement procedure is not given in the sources.
    pass

def render_mfr(scene_data, camera):
    """Executes multi-frame rate rendering using image-based warping for low latency [S1]."""
    # Fast node renders low-latency frame; slow nodes act as frame source [S1]
    # The exact warping implementation is not given in the sources.
    pass

def render_sfr_parallel(scene_data, camera):
    """Executes single-frame rate parallel raycasting to utilize all GPUs and avoid warping artifacts [S1]."""
    # Sort-first or sort-last parallel volume raycasting [S1]
    # The exact parallelization strategy is not given in the sources.
    pass

def main_loop(sampling_rate, resolution, other_params, last_params):
    if check_params_changed(other_params, last_params):
        recompute_decision_function(other_params)
    mfr_fps, sfr_fps = evaluate_decision_function(sampling_rate, resolution)
    method = 'MFR' if mfr_fps > sfr_fps else 'SFR'
    if method == 'MFR':
        render_mfr(scene_data, camera)
    else:
        render_sfr_parallel(scene_data, camera)
```

## Operating characteristics
*   **Time Complexity:** The decision function evaluation has virtually no performance impact [S1]. The full evaluation of the decision function takes approximately 2 minutes, but a good approximation can be obtained within 10 seconds [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** In tests with the MANIX dataset (256x256x230 voxels), multi-frame rate rendering was superior when volume rendering complexity was high, while sort-first parallel raycasting was superior when complexity was low [S1]. The performance was measured for sampling rates ranging from 6x undersampling to 6x oversampling compared to Nyquist rate sampling [S1].
*   **Scaling:** The system is designed for multi-GPU setups where a fast node handles low-latency rendering and slow nodes act as frame sources [S1].
*   **Strengths:** Provides stable frame rates and low latency to user input using multi-frame rate approach in combination with image-based rendering techniques [S1]. Avoids image warping artifacts when converging to single-frame rate rendering [S1].
*   **Limitations:** When converging to single-frame rate rendering, additional GPUs become useless as raycasting is performed entirely by the fast node [S1]. Requires periodic re-evaluation of the decision function if parameters change [S1].
*   **When to use:** Use when volume rendering complexity is high and low latency is critical [S1]. Use single-frame rate parallel raycasting when complexity is low to utilize all available resources [S1].

## Background
Dynamic Multi-Frame Rate Rendering was introduced by S. Hauswiesner, D. Kalkofen, and D. Schmalstieg in 2010 [S1]. It is part of a broader effort to display transparent volumetric scenes at high frame rates and little latency to user input [S1]. The method builds on the principles of multi-frame rate rendering and parallel volume raycasting [S1].

## Sources
[S1] Multi-Frame Rate Volume Rendering, https://doi.org/10.2312/egpgv.egpgv10.019-026, fetched 2026-10-29
