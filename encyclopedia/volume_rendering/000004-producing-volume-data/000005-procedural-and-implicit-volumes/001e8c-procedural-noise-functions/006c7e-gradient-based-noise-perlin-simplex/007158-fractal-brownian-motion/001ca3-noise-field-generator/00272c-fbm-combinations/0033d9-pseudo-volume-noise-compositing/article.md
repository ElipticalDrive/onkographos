# Pseudo-Volume Noise Compositing


## Overview
Pseudo-Volume Noise Compositing is a procedural noise generation technique that synthesizes volumetric-like textures by compositing four layered noise samples over a pseudo-3D position field derived from 2D UV coordinates [S1, unofficial]. It distinguishes itself from standard 2D noise by introducing a depth-like dimension via tiled cell indexing and applying a multiplicative cell mask to simulate volume boundaries, while using a power-law brightness reduction to control luminance [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Fragment UV coordinates ($uv$), `numCells` (integer, number of cells in x and y), `tilingPerCell` (float, repetitions per cell), `positionOffset` (vec3), `totalBrightness` (float), `cellMask` (float, output from position generator), `noiseLayerDistortion` (vec4), `noiseLayerEnabled` (vec4), `noiseLayerInverted` (vec4), and `noiseLayerN` (vec4 for each layer $N=1..4$, containing scale, power, angleOffset, type) [S1, unofficial].
*   **Output:** Final grayscale intensity $C$ with full alpha [S1, unofficial].

**Data Structures:**
*   `noiseLayerData` (vec4): Layout is `[scale, power, angleOffset, type]` [S1, unofficial].
    *   `scale` (float): Multiplier for noise position [S1, unofficial].
    *   `power` (float): Exponent for the noise value [S1, unofficial].
    *   `angleOffset` (float): Parameter for specific noise types (e.g., Voronoi) [S1, unofficial].
    *   `type` (int): 0=Perlin (`cnoise`), 1=Simplex (`snoise`), 2=Voronoi, 3=Nebula [S1, unofficial].
*   `noiseLayerDistortion` (vec4): Distortion strength for each layer [S1, unofficial].

**Steps:**

1.  **Compute Pseudo-Volume Position and Mask:**
    Remap UV to ensure correct stacking order: $uv_{remapped} = (uv.x, 1.0 - uv.y)$ [S1, unofficial].
    Calculate cell coordinates: $cells = fract(uv_{remapped} \cdot numCells)$ and $coords = uv_{remapped} \cdot numCells$ [S1, unofficial].
    Determine 1D frame index: $frameIndex1D = coords.y \cdot numCells + coords.x$ [S1, unofficial].
    Calculate depth index: $zFrameIndex = (frameIndex1D / (numCells \cdot numCells)) \cdot tilingPerCell$ [S1, unofficial].
    Calculate per-cell UV: $uv_{perCell} = cells \cdot tilingPerCell$ [S1, unofficial].
    Construct noise position: $noisePosition = (uv_{perCell}.x, uv_{perCell}.y, zFrameIndex) + positionOffset$ [S1, unofficial].
    Compute `cellMask` based on boundary conditions (e.g., hiding first/last cells to avoid artifacts) [S1, unofficial].

2.  **Transform Noise Position:**
    Apply additional UV modifications to `noisePosition` to produce `noiseSamplePosition` [S1, unofficial].

3.  **Sample Four Noise Layers:**
    For each layer $i \in \{1, 2, 3, 4\}$:
    *   Extract `noiseLayerData` for layer $i$ [S1, unofficial].
    *   If `noiseLayerEnabled[i]` is false, set $P_i = 1.0$ and continue [S1, unofficial].
    *   Calculate distortion strength: $distortionStrength = 0.1 \cdot noiseLayerDistortion[i]$ [S1, unofficial].
    *   If $distortionStrength > 0$, compute distortion noise $d = cnoise(distortionStrength \cdot noiseSamplePosition)$ and update $noiseSamplePosition \leftarrow noiseSamplePosition + d$ [S1, unofficial].
    *   Evaluate noise type:
        *   Type 0: $val = cnoise(noiseSamplePosition \cdot scale)$ [S1, unofficial].
        *   Type 1: $val = snoise(noiseSamplePosition \cdot scale)$ [S1, unofficial].
        *   Type 2: $val = voronoi(noiseSamplePosition \cdot scale, angleOffset, scale)$ [S1, unofficial].
        *   Type 3: $val = nebulaNoise(noiseSamplePosition \cdot scale)$ [S1, unofficial].
    *   Clamp value: $val = clamp(val, 0.0, 1.0)$ [S1, unofficial].
    *   Invert if `noiseLayerInverted[i]` is true: $val = 1.0 - val$ [S1, unofficial].
    *   Apply power: $P_i = val^{|power|}$ [S1, unofficial].

4.  **Sum Layers:**
    $Sum = P_1 + P_2 + P_3 + P_4$ [S1, unofficial].

5.  **Apply Brightness Reduction:**
    $Brightness = Sum^2 \cdot totalBrightness$ [S1, unofficial].

6.  **Compute Multiplicative Mask:**
    $Product = P_1 \cdot P_2 \cdot P_3 \cdot P_4$ [S1, unofficial].
    $Mask = Product \cdot cellMask$ [S1, unofficial].

7.  **Final Compositing:**
    $C = Brightness \cdot Mask$ [S1, unofficial].

**Formula:**
$$C = (P_1 + P_2 + P_3 + P_4)^2 \cdot B \cdot (P_1 \cdot P_2 \cdot P_3 \cdot P_4 \cdot M)$$
Where $P_i$ are the processed noise layer outputs, $B$ is `totalBrightness`, and $M$ is `cellMask` [S1, unofficial].

**Parameters:**
*   `totalBrightness`: Scalar uniform. Value not specified in sources [S1, unofficial].
*   `numCells`: Integer. Controls the grid density [S1, unofficial].
*   `tilingPerCell`: Float. Controls the frequency of noise within each cell [S1, unofficial].
*   `noiseLayerDistortion`: vec4. Controls the intensity of positional distortion per layer [S1, unofficial].
*   `noiseLayerEnabled`: vec4. Boolean flags (0 or >0) enabling/disabling layers [S1, unofficial].
*   `noiseLayerInverted`: vec4. Boolean flags (0 or >0) inverting layer outputs [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Boundary Artifacts:** Engines like Unity may produce artifacts at the edges of stacked noise slices. The implementation includes logic to hide the first and last cells (`isFirstCell`, `isLastCell`) to create padding [S1, unofficial].
*   **Precision:** The use of `fract` and large `numCells` values may introduce floating-point precision issues in the depth index calculation [S1, unofficial].
*   **Division by Zero:** If `numCells` is 0, the depth index calculation will fail. Not given in the sources.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pseudo_volume_noise_compositing(uv, num_cells, tiling_per_cell, position_offset, total_brightness, cell_mask, noise_layer_distortion, noise_layer_enabled, noise_layer_inverted, noise_layer_data_list):
    """
    uv: vec2 (fragment UV)
    num_cells: int
    tiling_per_cell: float
    position_offset: vec3
    total_brightness: float
    cell_mask: float
    noise_layer_distortion: vec4
    noise_layer_enabled: vec4
    noise_layer_inverted: vec4
    noise_layer_data_list: list of 4 vec4 [scale, power, angleOffset, type]
    """
    # 1. Compute Pseudo-Volume Position
    uv_remapped = vec2(uv.x, 1.0 - uv.y)
    cells = fract(uv_remapped * num_cells)
    coords = uv_remapped * num_cells
    
    frame_index_1d = coords.y * num_cells + coords.x
    z_frame_index = (frame_index_1d / (num_cells * num_cells)) * tiling_per_cell
    
    uv_per_cell = cells * tiling_per_cell
    noise_position = vec3(uv_per_cell.x, uv_per_cell.y, z_frame_index) + position_offset
    
    # Note: cell_mask is assumed to be computed externally or via createPseudoVolumePosition
    # Logic for hiding first/last cells is embedded in cell_mask generation
    
    # 2. Transform Noise Position (modify_UV_for_NoiseSamplePosition)
    # Standard coordinate space transform [G]
    noise_sample_position = modify_uv_for_noise_sample_position(noise_position)
    
    sampled_noises = vec4(1.0)
    
    # 3. Sample Four Noise Layers
    for i in range(4):
        if not noise_layer_enabled[i]:
            sampled_noises[i] = 1.0
            continue
            
        noise_data = noise_layer_data_list[i]
        scale = noise_data.x
        power = noise_data.y
        angle_offset = noise_data.z
        noise_type = int(noise_data.w)
        
        distortion_strength = 0.1 * noise_layer_distortion[i]
        current_pos = noise_sample_position
        
        if distortion_strength > 0:
            # Standard 3D Perlin noise evaluation [G]
            distortion_noise = perlin_noise(distortion_strength * current_pos)
            current_pos += distortion_noise
            
        # Evaluate noise type
        if noise_type == 0:
            val = perlin_noise(current_pos * scale)
        elif noise_type == 1:
            val = simplex_noise(current_pos * scale)
        elif noise_type == 2:
            val = voronoi_noise(current_pos * scale, angle_offset, scale)
        elif noise_type == 3:
            val = nebula_noise(current_pos * scale)
        else:
            val = 0.0
            
        val = clamp(val, 0.0, 1.0)
        
        if noise_layer_inverted[i]:
            val = 1.0 - val
            
        power_abs = abs(power)
        sampled_noises[i] = pow(val, power_abs)
        
    p1, p2, p3, p4 = sampled_noises.x, sampled_noises.y, sampled_noises.z, sampled_noises.w
    
    # 4. Sum Layers
    sum_val = p1 + p2 + p3 + p4
    
    # 5. Apply Brightness Reduction
    brightness = pow(sum_val, 2.0) * total_brightness
    
    # 6. Compute Multiplicative Mask
    product = p1 * p2 * p3 * p4
    mask = product * cell_mask
    
    # 7. Final Compositing
    c = brightness * mask
    
    return c

def perlin_noise(p):
    # Standard 3D Perlin noise evaluation [G]
    return ...

def simplex_noise(p):
    # Standard 3D Simplex noise evaluation [G]
    return ...

def voronoi_noise(p, angle_offset, scale):
    # Standard 3D Voronoi noise evaluation [G]
    return ...

def nebula_noise(p):
    # Standard 3D Nebula noise evaluation [G]
    return ...

def modify_uv_for_noise_sample_position(pos):
    # Standard coordinate space transform [G]
    return pos
```

## Operating characteristics
*   **Time Complexity:** O(1) per fragment, as it involves a fixed number of noise evaluations (4 layers) and arithmetic operations [S1, unofficial].
*   **Memory Complexity:** O(1), requiring only uniform storage for layer parameters [S1, unofficial].
*   **Strengths:** Produces complex, volumetric-like textures from 2D inputs; allows fine-grained control over each layer's type, distortion, and inversion; multiplicative masking allows for organic voids or cell structures [S1, unofficial].
*   **Limitations:** The "pseudo-3D" nature is an approximation; true volumetric lighting or ray-marching is not supported. Performance scales linearly with the number of layers, but the current implementation is fixed at 4 [S1, unofficial].
*   **When to use:** When a 2D shader needs to simulate volumetric noise effects (e.g., smoke, clouds, stone textures) without the computational cost of true 3D noise or ray-marching [S1, unofficial].
*   **When not to use:** When true 3D consistency is required across different viewing angles or when dynamic lighting within the volume is needed [S1, unofficial].

## Background
Pseudo-Volume Noise Compositing is implemented in the "texturinator" project by MangoButtermilch, an unofficial tool for generating procedural textures [S1, unofficial]. It leverages GLSL noise functions (`cnoise`, `snoise`, `voronoi`, `nebulaNoise`) to create layered noise patterns. The technique is designed to work within fragment shaders, mapping 2D UVs to a pseudo-3D space to simulate depth [S1, unofficial].

## Sources
[S1] texturinator, https://github.com/MangoButtermilch/texturinator, fetched 2026-10-26 (unofficial)
