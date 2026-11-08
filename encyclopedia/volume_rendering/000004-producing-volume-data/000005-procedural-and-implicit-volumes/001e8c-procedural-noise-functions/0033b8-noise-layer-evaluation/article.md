# Noise Layer Evaluation

Also called: Procedural Noise Layer Evaluation

## Overview
Noise Layer Evaluation is a procedural shading technique that computes a single layer of volumetric density or color by sampling a base noise function (such as Perlin, Simplex, or Worley) at a distorted position, applying non-linear power scaling, and optionally inverting the result [S1, unofficial]. It serves as a modular building block for complex volumetric effects, allowing artists to stack multiple layers with independent scales, powers, and noise types to create detailed textures like clouds, marble, or terrain [S1, unofficial]. Unlike single-function noise generators, this method encapsulates distortion, type selection, and post-processing into a unified evaluation step, distinguishing it from siblings like [Worley Noise](1726-worley-noise.md) or [Gradient-based Noise (Perlin/Simplex)](3860-gradient-based-noise-perlin-simplex.md) by adding configurable layer parameters and positional warping [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Input**: A 3D sample position $P_{sample}$, a configuration vector $L_{data}$ encoding layer parameters, a distortion strength scalar $D_{str}$, and boolean flags for inversion ($I_{vert}$) and enablement ($E_{nabled}$).
- **Output**: A scalar value $V_{final}$ representing the layer's contribution to density or color.

**Data Structures**
The layer configuration is packed into a 4-component vector $L_{data} = (x, y, z, w)$ [S1, unofficial]:
- $x$: Noise scale factor ($S_{cale}$).
- $y$: Noise power exponent ($P_{ower}$).
- $z$: Angle offset ($A_{offset}$), used primarily for Voronoi-based noise.
- $w$: Noise type identifier ($T_{ype}$), an integer mapping to specific noise functions.

**Numbered Steps**
1. **Enable Check**: If $E_{nabled}$ is false, return $V_{final} = 1.0$ immediately [S1, unofficial].
2. **Position Distortion**: Calculate distortion magnitude $M_{ag} = 0.1 \times D_{str}$ [S1, unofficial]. If $M_{ag} > 0$, compute a distortion vector $D_{vec}$ by sampling a continuous noise function (e.g., cnoise) at position $P_{sample} \times M_{ag}$. Update the sample position: $P_{distorted} = P_{sample} + D_{vec}$ [S1, unofficial]. If $M_{ag} \le 0$, $P_{distorted} = P_{sample}$ [S1, unofficial].
3. **Parameter Unpacking**: Extract $S_{cale} = L_{data}.x$, $P_{ower} = L_{data}.y$, $A_{offset} = L_{data}.z$, and $T_{ype} = \lfloor L_{data}.w \rfloor$ [S1, unofficial].
4. **Noise Sampling**: Initialize $V_{sample} = 0.0$. Evaluate the base noise function $N(P)$ based on $T_{ype}$:
   - If $T_{ype} = 0$: $V_{sample} = \text{cnoise}(P_{distorted} \times S_{cale})$ [S1, unofficial].
   - If $T_{ype} = 1$: $V_{sample} = \text{snoise}(P_{distorted} \times S_{cale})$ [S1, unofficial].
   - If $T_{ype} = 2$: $V_{sample} = \text{voronoi}(P_{distorted} \times S_{cale}, A_{offset}, S_{cale})$ [S1, unofficial].
   - If $T_{ype} = 3$: $V_{sample} = \text{nebulaNoise}(P_{distorted} \times S_{cale})$ [S1, unofficial].
   - Otherwise: $V_{sample} = 0.0$ [S1, unofficial].
5. **Clamping and Inversion**: Clamp $V_{sample}$ to $[0.0, 1.0]$ to get $V_{clamped}$ [S1, unofficial]. If $I_{vert}$ is true, set $V_{val} = 1.0 - V_{sample}$ (using the unclamped value) [S1, unofficial]. Else, set $V_{val} = V_{clamped}$ [S1, unofficial].
6. **Power Scaling**: Compute $V_{final} = |V_{val}|^{P_{ower}}$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_noise_layer(position: vec3, layer_data: vec4, distortion_strength: float, invert: bool, enabled: bool) -> float:
    # Step 1: Enable Check
    if not enabled:
        return 1.0

    # Step 2: Position Distortion
    mag = 0.1 * distortion_strength
    if mag > 0:
        # Assuming cnoise is available as a base noise function
        distortion_vec = vec3(cnoise(position * mag))
        sample_pos = position + distortion_vec
    else:
        sample_pos = position

    # Step 3: Parameter Unpacking
    scale = layer_data.x
    power = layer_data.y
    angle_offset = layer_data.z
    noise_type = int(layer_data.w)

    # Step 4: Noise Sampling
    sampled_value = 0.0
    scaled_pos = sample_pos * scale

    if noise_type == 0:
        sampled_value = cnoise(scaled_pos)
    elif noise_type == 1:
        sampled_value = snoise(scaled_pos)
    elif noise_type == 2:
        sampled_value = voronoi(scaled_pos, angle_offset, scale)
    elif noise_type == 3:
        sampled_value = nebula_noise(scaled_pos)
    else:
        sampled_value = 0.0

    # Step 5: Clamping and Inversion
    clamped_value = clamp(sampled_value, 0.0, 1.0)
    
    if invert:
        # Pitfall: Inversion uses unclamped sampled_value, not clamped_value
        val = 1.0 - sampled_value
    else:
        val = clamped_value

    # Step 6: Power Scaling
    final_value = abs(val) ** power
    
    return final_value
```

**Parameters and Typical Values**
- `noiseLayerData` (vec4): Encoded parameters. $x \in [0.0, \infty)$ for scale, $y \in [0.0, \infty)$ for power, $z$ for angle offset, $w \in \{0, 1, 2, 3\}$ for type [S1, unofficial].
- `positionDistortion` (float): Controls the intensity of positional warping. Typical values are small (e.g., 1.0–5.0) to avoid excessive distortion [S1, unofficial].
- `invert` (bool): Toggles inversion logic [S1, unofficial].
- `enabled` (bool): Enables/disables the layer [S1, unofficial].
- `densityMult` (float): Default 1.0, range $[0.0, 1.0]$, step 0.001. Multiplies the final density in the raymarching loop [S1, unofficial].
- `maxSteps` (int): Default 70, range $[40, 256]$ [S1, unofficial].
- `stepSize` (float): Default 0.01, range $[0.00001, 0.1]$ [S1, unofficial].

**Edge Cases and Pitfalls**
- **Inversion Clamping Bug**: The inversion branch uses the raw `sampledNoiseValue` instead of the clamped `value`. This means if `invert` is true, the clamp applied in the previous step is ignored, potentially allowing values outside $[0.0, 1.0]$ to be inverted and raised to a power [S1, unofficial].
- **Unused Depth Parameter**: The function signature may include a `depth` parameter that is declared but never referenced in the body, leading to unused computation or confusion [S1, unofficial].
- **Division by Zero**: If `scale` is 0.0, noise sampling may behave unpredictably depending on the implementation of the base noise functions [G].

## Operating characteristics
- **Time Complexity**: $O(1)$ per layer evaluation, assuming constant-time noise function evaluation. The cost depends on the complexity of the base noise function (e.g., Voronoi is generally more expensive than Simplex) [S1, unofficial].
- **Memory Complexity**: $O(1)$, as it computes values on-the-fly without storing large lookup tables [S1, unofficial].
- **Strengths**: Modular design allows for complex effects by stacking layers; supports multiple noise types; distortion adds organic variation [S1, unofficial].
- **Limitations**: The inversion bug can lead to unexpected artifacts; distortion adds computational overhead; power scaling can amplify noise artifacts if not clamped correctly [S1, unofficial].
- **When to Use**: Ideal for procedural volumetric texturing where artistic control over scale, power, and noise type is required [S1, unofficial].
- **When Not to Use**: When performance is critical and distortion is unnecessary; when strict clamping is required before inversion [S1, unofficial].

## Background
Noise Layer Evaluation originates from procedural texture generation techniques used in real-time rendering, particularly in WebGL-based volume visualization tools like Texturinator [S1, unofficial]. It builds upon foundational noise functions such as Perlin noise, Simplex noise, and Worley noise, extending them with layer-specific parameters for scale, power, and distortion [S1, unofficial]. The method is commonly used in 3D volume preview services and shader-based rendering pipelines to generate complex, organic textures for clouds, terrain, and materials [S1, unofficial].

## Sources
[S1] MangoButtermilch/texturinator, https://github.com/MangoButtermilch/texturinator, fetched 2026-10-26 (unofficial)
