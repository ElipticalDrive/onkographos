# Field Output Prediction

Also called: Model Output Unpacking

## Overview
Field Output Prediction is the final stage of the RGB Network Layer in neural volume rendering, responsible for passing input positions through the trained neural network model and unpacking the raw tensor outputs into semantically meaningful values based on the specified reconstruction target [S1, unofficial]. It serves as the bridge between the latent neural representation and the physical quantities (such as density, temperature, or RGB color) required for subsequent volume rendering steps [S1, unofficial]. Unlike the Sigma Network Layer, which focuses specifically on opacity/density prediction, this method handles general field outputs and supports various target configurations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tensor of query positions `x_AABB` (typically shape `[N, D]` where `N` is the number of points and `D` is the spatial dimensionality, e.g., 3) and a `reconstruction_target` string or enum defining the desired output format [S1, unofficial].
*   **Output:** Unpacked field values (e.g., `log_ne`, `log_temp`, RGB, etc.) structured according to the `reconstruction_target` [S1, unofficial].

**Data Structures:**
*   `self.model`: The neural network (MLP) instance.
*   `self.reconstruction_target`: Configuration specifying which fields to extract and how to format them.
*   `raw`: The intermediate tensor output from the model before unpacking.

**Steps:**
1.  **Forward Pass:** Compute the raw output tensor by passing the input positions through the neural network model: `raw = self.model(x_AABB)` [S1, unofficial].
2.  **Dimension Handling:** Check if the output tensor `raw` has one fewer dimension than the input `x_AABB`. If `raw.ndim == x_AABB.ndim - 1`, append a singleton dimension to the last axis to maintain batch/point consistency: `raw = raw[..., None]` [S1, unofficial].
3.  **Reshape:** Flatten the batch and spatial dimensions into a single dimension while preserving the feature channels: `raw_flat = raw.reshape(-1, raw.shape[-1])` [S1, unofficial].
4.  **Unpack:** Pass the flattened raw tensor and the reconstruction target to the unpacking function: `return unpack_plasma_outputs(raw_flat, self.reconstruction_target)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def field_output_prediction(self, x_AABB, reconstruction_target):
    """
    Passes input positions through the neural network model and unpacks the raw outputs.
    
    Args:
        x_AABB: Tensor of query positions, shape (N, D).
        reconstruction_target: String or enum specifying the output format.
        
    Returns:
        Unpacked field values based on reconstruction_target.
    """
    # Step 1: Compute raw output from the model
    raw = self.model(x_AABB)
    
    # Step 2: Handle dimensionality mismatch
    # If raw has one less dimension than input, add a trailing dimension
    if raw.ndim == x_AABB.ndim - 1:
        raw = raw[..., None]
        
    # Step 3: Reshape to (num_points, num_features)
    raw_flat = raw.reshape(-1, raw.shape[-1])
    
    # Step 4: Unpack based on target
    return unpack_plasma_outputs(raw_flat, reconstruction_target)
```

**Parameters and Edge Cases:**
*   **Dimensionality Check:** The check `raw.ndim == x_AABB.ndim - 1` handles cases where the network reduces spatial dimensions (e.g., outputting a scalar per point instead of a vector) [S1, unofficial].
*   **Reconstruction Target:** The specific fields extracted depend on `reconstruction_target`. Common targets include density (`ne`), temperature (`temp`), or RGB color [S1, unofficial].
*   **Precision:** Ensure `x_AABB` is in the correct coordinate space (e.g., AABB-normalized) expected by `self.model` [S1, unofficial].

## Subtopics

### Learned Fields Prediction with Clamping
Predicts field outputs from the model and applies clamping to log density (`log_ne`) and log temperature (`log_temp`) based on configured bounds [S1, unofficial]. This variant ensures physical plausibility by restricting values to valid ranges derived from empirical data (e.g., PSI and Chianti constraints) [S1, unofficial].
*   **Definition:** Extracts `log_ne` and `log_temp` from the model output and clamps them if `self.clamp` is True [S1, unofficial].
*   **Steps:**
    1.  Call `self._predict_field_outputs(x_AABB)` to get `outs` containing `log_ne` and `log_temp` [S1, unofficial].
    2.  Extract `log_ne = outs.log_ne`. If `self.clamp` is True, clamp `log_ne` to `[self.log_ne_min, self.log_ne_max]` [S1, unofficial].
    3.  Extract `log_temp = outs.log_temp`. If `log_temp` is not None and `self.clamp` is True, clamp `log_temp` to `[self.log_temp_min, self.log_temp_max]` [S1, unofficial].
*   **Layout:** Returns `outs`, `log_ne`, `log_temp` [S1, unofficial].
*   **When to choose:** Use when rendering physical fields (e.g., plasma density/temperature) where values must adhere to known physical bounds [S1, unofficial].

### Raw to Outputs Conversion
Transforms raw network predictions into semantically meaningful values like RGB, depth, and weights for volume rendering [S2, unofficial]. This variant is tailored for standard neural radiance fields (NeRF) where the goal is to produce photorealistic images [S2, unofficial].
*   **Definition:** Converts raw RGB and sigma outputs into alpha, weights, RGB map, depth map, disparity map, and accumulated alpha map [S2, unofficial].
*   **Steps:**
    1.  Extract RGB values from the first three channels of raw output [S2, unofficial].
    2.  Convert raw sigma to alpha and weights using `raw2alpha` [S2, unofficial].
    3.  Compute RGB map by summing weights multiplied by RGB values [S2, unofficial].
    4.  Compute depth map by summing weights multiplied by z_vals [S2, unofficial].
    5.  Compute disparity map as the inverse of depth, with a minimum value to avoid division by zero [S2, unofficial].
    6.  Compute accumulated alpha map by summing weights [S2, unofficial].
    7.  If white background is requested, add the difference between 1 and accumulated alpha to the RGB map [S2, unofficial].
*   **When to choose:** Use for standard NeRF-based rendering where RGB, depth, and alpha compositing are required [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by the forward pass of the neural network `self.model(x_AABB)`, which is $O(N \cdot L \cdot W)$ where $N$ is the number of points, $L$ is the number of layers, and $W$ is the width [S1, unofficial]. The unpacking step is $O(N \cdot F)$ where $F$ is the number of output features [S1, unofficial].
*   **Memory Complexity:** $O(N \cdot F)$ to store the raw output tensor before unpacking [S1, unofficial].
*   **Strengths:** Flexible output format via `reconstruction_target`; handles dimensionality mismatches automatically [S1, unofficial].
*   **Limitations:** Performance is bottlenecked by the neural network inference; unpacking logic is specific to the implementation of `unpack_plasma_outputs` [S1, unofficial].
*   **When to use:** When the neural network output needs to be converted into specific physical or visual fields for rendering [S1, unofficial].
*   **When not to use:** If the network output is already in the desired format or if a different post-processing step is required [S1, unofficial].

## Background
Field Output Prediction is a component of neural volume rendering pipelines that emerged with the rise of neural implicit representations for 3D scenes [S1, unofficial]. It was formalized in implementations such as CoroNeRF, where it handles the unpacking of plasma-specific fields (density, temperature) from neural network outputs [S1, unofficial]. The method builds on the general NeRF framework, where raw network outputs are transformed into rendering-compatible values [S2, unofficial].

## Sources
[S1] AlanHsu314/CoroNeRF_public, https://github.com/AlanHsu314/CoroNeRF_public, fetched 2026-10-27 (unofficial)
[S2] Xharlie/pointnerf, https://github.com/Xharlie/pointnerf, fetched 2026-10-26 (unofficial)
