# Noise Layer Evaluation

Also called: Procedural Noise Layer Evaluation

## Overview
Noise Layer Evaluation is a procedural shading technique that computes a single noise layer value for volume or terrain rendering, supporting multiple noise primitives, positional distortion, power scaling, and inversion [S1, unofficial]. It differs from sibling methods by bundling noise type selection, distortion, and post-processing into a unified evaluation step that can be toggled or inverted independently, whereas siblings like [Worley Noise](1726-worley-noise.md) or [Gradient-based Noise (Perlin/Simplex)](3860-gradient-based-noise-perlin-simplex.md) typically focus on the generation of the noise field itself without this specific layer-compositing logic.

## How it works
**Inputs and Outputs**
*   **Input:** `noiseSamplePosition` (vec3), `noiseLayerData` (vec4), `positionDistortion` (float), `invert` (bool), `enabled` (bool).
*   **Output:** `finalValue` (float).

**Data Structures**
*   `noiseLayerData`: A 4-component vector encoding layer parameters:
    *   `x`: `noiseScale` (float)
    *   `y`: `noisePower` (float)
    *   `z`: `angleOffset` (float)
    *   `w`: `noiseType` (integer, cast from float)

**Numbered Steps**
1.  **Enable Check:** If `enabled` is false, set `finalValue` to 1.0 and return immediately [S1, unofficial].
2.  **Distortion Calculation:** Compute `distortionStrength` as $0.1 \times \text{positionDistortion}$. If `distortionStrength` > 0, sample a noise function `cnoise` at `distortionStrength * noiseSamplePosition`. Add the resulting 3D vector to `noiseSamplePosition` to create a distorted position [S1, unofficial].
3.  **Parameter Unpacking:** Extract `noiseScale`, `noisePower`, `angleOffset`, and `noiseType` from `noiseLayerData` [S1, unofficial].
4.  **Noise Sampling:** Initialize `sampledNoiseValue` to 0.0. Select the noise function based on `noiseType`:
    *   0: `cnoise` (Curl Noise)
    *   1: `snoise` (Simplex Noise)
    *   2: `voronoi` (Voronoi Noise)
    *   3: `nebulaNoise`
    *   Evaluate the selected function at `noiseSamplePosition * noiseScale`. For Voronoi, pass `angleOffset` and `noiseScale` as additional arguments [S1, unofficial].
5.  **Clamping and Inversion:** Clamp `sampledNoiseValue` to the range $[0.0, 1.0]$ and store in `value`. If `invert` is true, overwrite `value` with $1.0 - \text{sampledNoiseValue}$ (note: this bypasses the previous clamp) [S1, unofficial].
6.  **Power Scaling:** Compute `absNoisePower` as $|\text{noisePower}|$. Calculate `finalValue` as $\text{value}^{\text{absNoisePower}}$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_noise_layer(noise_sample_position, noise_layer_data, position_distortion, invert, enabled):
    # Step 1: Enable Check
    if not enabled:
        return 1.0

    # Step 2: Distortion
    distortion_strength = 0.1 * position_distortion
    if distortion_strength > 0:
        # cnoise is assumed to be a defined curl noise function
        distortion_offset = cnoise(distortion_strength * noise_sample_position)
        distorted_position = noise_sample_position + distortion_offset
    else:
        distorted_position = noise_sample_position

    # Step 3: Unpack Parameters
    noise_scale = noise_layer_data.x
    noise_power = noise_layer_data.y
    angle_offset = noise_layer_data.z
    noise_type = int(noise_layer_data.w)

    # Step 4: Noise Sampling
    sampled_noise_value = 0.0
    scaled_position = distorted_position * noise_scale

    if noise_type == 0:
        sampled_noise_value = cnoise(scaled_position)
    elif noise_type == 1:
        sampled_noise_value = snoise(scaled_position)
    elif noise_type == 2:
        sampled_noise_value = voronoi(scaled_position, angle_offset, noise_scale)
    elif noise_type == 3:
        sampled_noise_value = nebula_noise(scaled_position)
    else:
        # Fallback for unknown types
        sampled_noise_value = 0.0

    # Step 5: Clamping and Inversion
    value = clamp(sampled_noise_value, 0.0, 1.0)
    
    if invert:
        # Pitfall: Inversion uses original sampled value, bypassing clamp
        value = 1.0 - sampled_noise_value

    # Step 6: Power Scaling
    abs_noise_power = abs(noise_power)
    final_value = pow(value, abs_noise_power)

    return final_value
```

**Parameters**
*   `noiseLayerData` (vec4): Encoded layer parameters. `x`=scale, `y`=power, `z`=angle offset, `w`=noise type (integer 0-3) [S1, unofficial].
*   `positionDistortion` (float): Scales the offset applied to the sample position [S1, unofficial].
*   `invert` (bool): When true, inverts the noise value before power application [S1, unofficial].
*   `enabled` (bool): When false, forces the layer output to 1.0 [S1, unofficial].
*   `densityMult` (float): Default 1.0, range $[0.0, 1.0]$, step 0.001. Multiplies volume density in the raymarching loop [S1, unofficial].
*   `maxSteps` (int): Default 70, range $[40, 256]$, step 1. Maximum number of raymarching steps per ray [S1, unofficial].
*   `stepSize` (float): Default 0.01, range $[0.00001, 0.1]$, step 0.000001. Distance between consecutive sample positions along the ray [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Inversion Bypass:** The inversion branch uses `sampledNoiseValue` instead of the clamped `value`. This means if `invert` is true, the clamp applied in Step 5 is ignored, potentially allowing values outside $[0.0, 1.0]$ to be inverted and raised to a power [S1, unofficial].
*   **Unused Depth Parameter:** The function signature may include a `depth` parameter which is declared but never referenced or used in the function body [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per layer evaluation, assuming constant-time noise primitives.
*   **Memory Complexity:** $O(1)$ auxiliary memory.
*   **Strengths:** Unified handling of multiple noise types and post-processing effects (distortion, inversion, power) in a single pass.
*   **Limitations:** The inversion logic bug (bypassing clamp) can lead to unexpected artifacts if not accounted for. Performance depends heavily on the underlying noise primitives (`cnoise`, `snoise`, etc.).
*   **When to use:** When a procedural volume requires flexible noise layering with distortion and non-linear scaling.
*   **When not to use:** When strict bounded output $[0.0, 1.0]$ is required regardless of inversion settings.

## Background
This method is derived from the procedural noise layer evaluation logic found in the Texturinator project, an unofficial implementation for 3D volume previewing [S1, unofficial]. It integrates standard gradient-based noise primitives (like Simplex and Curl noise) with a specific layer management system used in real-time volume rendering applications.

## Sources
[S1] MangoButtermilch/texturinator, https://github.com/MangoButtermilch/texturinator, fetched 2026-10-26 (unofficial)
