# Multi-Mode Volume Ray Casting

Also called: Custom Volume Ray Casting with Shading and Multiple Modes, MyVolumeRayCastCompositeFunction

## Overview
Multi-Mode Volume Ray Casting is a custom implementation of direct volume ray casting that extends the standard compositing approach to support three distinct rendering modes: Full (standard compositing with shading), Maximum Intensity Projection (MIP), and XRay (average intensity projection) [S1, unofficial]. Unlike standard compositing which relies on pre-multiplied alpha blending, this method explicitly calculates shaded color contributions and blends them with the background using a specific accumulation formula, while also supporting nearest-neighbor interpolation and Phong shading [S1, unofficial]. It is designed for scalar volume data and is implemented as a custom ray cast function within the VTK framework [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume dataset. The implementation supports only `VTK_UNSIGNED_CHAR` (8-bit) and `VTK_UNSIGNED_SHORT` (16-bit) data types [S1, unofficial].
*   **Input:** Rendering parameters including ray start position, ray increment, number of steps, and transfer functions (scalar-to-color and scalar-to-opacity) [S1, unofficial].
*   **Input:** Shading tables for diffuse and specular components (Red, Green, Blue channels) and encoded normals [S1, unofficial].
*   **Output:** Final pixel color $(R, G, B)$ and accumulated opacity $\alpha$ [S1, unofficial].

**Data Structures:**
The method operates on two primary information structures passed to the ray casting function:
1.  **`vtkVolumeRayCastDynamicInfo`**: Contains per-ray runtime data.
    *   `NumberOfStepsToTake` (int): Total steps to iterate.
    *   `TransformedStart` (float[3]): Ray origin in voxel space.
    *   `TransformedIncrement` (float[3): Step vector in voxel space.
    *   `Color` (float[4]): Output color $(R, G, B, \alpha)$.
    *   `NumberOfStepsTaken` (int): Actual steps executed (for early termination).
2.  **`vtkVolumeRayCastStaticInfo`**: Contains per-volume static data.
    *   `ScalarDataPointer` (void*): Pointer to the raw voxel data.
    *   `ScalarDataType` (enum): Data type identifier.
    *   `RedDiffuseShadingTable`, `GreenDiffuseShadingTable`, `BlueDiffuseShadingTable` (float*): Lookup tables for diffuse shading.
    *   `RedSpecularShadingTable`, `GreenSpecularShadingTable`, `BlueSpecularShadingTable` (float*): Lookup tables for specular shading.
    *   `EncodedNormals` (unsigned short*): Pre-computed normal vectors.
    *   `DataIncrement` (int[3]): Strides for x, y, z dimensions to calculate linear memory offset.
    *   `Volume` (vtkVolume*): Reference to the volume object containing transfer functions.

**Algorithm Steps:**

1.  **Initialization**:
    *   Retrieve `data_ptr` from `staticInfo->ScalarDataPointer`.
    *   Retrieve `num_steps`, `ray_start`, and `ray_increment` from `dynamicInfo`.
    *   Retrieve transfer functions `ScalarToOpacityTF` and `ScalarToColorTF` from `staticInfo->Volume`.
    *   Retrieve `xinc`, `yinc`, `zinc` from `staticInfo->DataIncrement`.
    *   Initialize ray position in voxel space: $\mathbf{p} = \mathbf{ray\_start}$.
    *   Initialize accumulators: $C_R = C_G = C_B = 0.0$, $\alpha_{acc} = 0.0$.
    *   Initialize MIP variables: $\alpha_{max} = 0.0$, $V_{max} = 0.0$.
    *   Initialize global maximum scalar value $V_{global\_max} = 0.0$ [S1, unofficial].

2.  **Ray Loop**:
    Iterate for $i$ from $0$ to $num\_steps - 1$:
    *   **Early Termination**: If $\alpha_{acc} \ge 1.0$, break the loop [S1, unofficial].
    *   **Voxel Coordinate Calculation**:
        Round the current ray position to the nearest integer to get voxel coordinates:
        $$
        v_x = \text{round}(p_x), \quad v_y = \text{round}(p_y), \quad v_z = \text{round}(p_z)
        $$
        [S1, unofficial]
    *   **Data Access**:
        Calculate linear offset:
        $$
        \text{offset} = v_z \cdot \text{zinc} + v_y \cdot \text{yinc} + v_x \cdot \text{xinc}
        $$
        Retrieve scalar value: $V = \text{data\_ptr}[\text{offset}]$ [S1, unofficial].
    *   **Global Maximum Update**:
        If $V > V_{global\_max}$, set $V_{global\_max} = V$ [S1, unofficial].

    *   **Mode-Specific Processing**:
        Let $\mathbf{c}_{raw} = (r, g, b)$ be the color from `ScalarToColorTF[V]` and $\alpha$ be the opacity from `ScalarToOpacityTF[V]`.

        *   **Mode MIP (`currentRenderMode == 0`)**:
            If $V > V_{max}$:
            $$
            V_{max} = V
            $$
            $$
            \mathbf{c}_{raw} = \text{ScalarToColorTF}[V]
            $$
            Set $\alpha_{acc} = 1.0 - \alpha$ [S1, unofficial].
            *(Note: The source sets accumulators to the new color multiplied by opacity, but effectively MIP replaces the accumulated color with the max intensity's color when a new max is found, and forces opacity to ensure the final pixel reflects this max)*. Specifically:
            $$
            C_R = \alpha \cdot r, \quad C_G = \alpha \cdot g, \quad C_B = \alpha \cdot b
            $$
            $$
            \alpha_{acc} = 1.0 - \alpha
            $$
            [S1, unofficial]

        *   **Mode XRay (`currentRenderMode == 1`)**:
            Accumulate scalar values into all color channels:
            $$
            C_R \leftarrow C_R + V
            $$
            $$
            C_G \leftarrow C_G + V
            $$
            $$
            C_B \leftarrow C_B + V
            $$
            [S1, unofficial]

        *   **Mode Full (`currentRenderMode == 2`)**:
            Lookup $\mathbf{c}_{raw} = (r, g, b)$ from `ScalarToColorTF[V]`.
            Compute shaded values (assuming shading tables are applied to $r, g, b$ based on normals, resulting in shaded $r, g, b$):
            $$
            r_{shaded} = \alpha \cdot r, \quad g_{shaded} = \alpha \cdot g, \quad b_{shaded} = \alpha \cdot b
            $$
            Accumulate color using the specific blending formula:
            $$
            C_R \leftarrow r_{shaded} \cdot (1.0 - \alpha_{acc}) + C_R
            $$
            $$
            C_G \leftarrow g_{shaded} \cdot (1.0 - \alpha_{acc}) + C_G
            $$
            $$
            C_B \leftarrow b_{shaded} \cdot (1.0 - \alpha_{acc}) + C_B
            $$
            Accumulate opacity:
            $$
            \alpha_{acc} \leftarrow \alpha \cdot (1.0 - \alpha_{acc}) + \alpha_{acc}
            $$
            [S1, unofficial]

    *   **Ray Advancement**:
        $$
        \mathbf{p} \leftarrow \mathbf{p} + \mathbf{ray\_increment}
        $$
        [S1, unofficial]

3.  **Finalization**:
    *   Cap all accumulated color intensities $C_R, C_G, C_B$ at $1.0$ [S1, unofficial].
    *   Write final color $(C_R, C_G, C_B)$ and opacity $\alpha_{acc}$ to `dynamicInfo->Color`.
    *   Write number of steps taken to `dynamicInfo->NumberOfStepsTaken` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cast_ray(data_ptr, scalar_data_type, dynamic_info, static_info, current_render_mode):
    # 1. Initialization
    num_steps = dynamic_info.NumberOfStepsToTake
    ray_start = dynamic_info.TransformedStart
    ray_increment = dynamic_info.TransformedIncrement
    xinc, yinc, zinc = static_info.DataIncrement
    scalar_to_color_tf = static_info.Volume.ScalarToColorTF
    scalar_to_opacity_tf = static_info.Volume.ScalarToOpacityTF
    
    # Accumulators
    accum_r = 0.0
    accum_g = 0.0
    accum_b = 0.0
    accum_alpha = 0.0
    
    # MIP variables
    max_val = 0.0
    max_color_r = 0.0
    max_color_g = 0.0
    max_color_b = 0.0
    
    # Global max for XRay normalization (assumed available globally or passed)
    global_max_val = get_global_max() 
    
    # Ray position
    pos_x = ray_start[0]
    pos_y = ray_start[1]
    pos_z = ray_start[2]
    
    # 2. Loop
    steps_taken = 0
    for i in range(num_steps):
        # Early termination
        if accum_alpha >= 1.0:
            break
            
        # Voxel coordinates (nearest neighbor)
        voxel_x = round(pos_x)
        voxel_y = round(pos_y)
        voxel_z = round(pos_z)
        
        # Data access
        offset = voxel_z * zinc + voxel_y * yinc + voxel_x * xinc
        scalar_val = data_ptr[offset]
        
        # Update global max
        if scalar_val > global_max_val:
            set_global_max(scalar_val)
            
        # Get transfer function values
        # Note: In the source, ScalarToColorTF returns RGB, ScalarToOpacityTF returns Alpha
        color_rgb = scalar_to_color_tf(scalar_val)
        r_raw, g_raw, b_raw = color_rgb
        alpha = scalar_to_opacity_tf(scalar_val)
        
        # Mode processing
        if current_render_mode == 0: # MIP
            if scalar_val > max_val:
                max_val = scalar_val
                max_color_r = r_raw
                max_color_g = g_raw
                max_color_b = b_raw
                accum_alpha = 1.0 - alpha
                accum_r = alpha * r_raw
                accum_g = alpha * g_raw
                accum_b = alpha * b_raw
                
        elif current_render_mode == 1: # XRay
            accum_r += scalar_val
            accum_g += scalar_val
            accum_b += scalar_val
            
        elif current_render_mode == 2: # Full
            # Apply shading (simplified: source implies shading tables modify r,g,b)
            # Assuming r_raw, g_raw, b_raw are already shaded or shading is a lookup
            r_shaded = alpha * r_raw
            g_shaded = alpha * g_raw
            b_shaded = alpha * b_raw
            
            accum_r = r_shaded * (1.0 - accum_alpha) + accum_r
            accum_g = g_shaded * (1.0 - accum_alpha) + accum_g
            accum_b = b_shaded * (1.0 - accum_alpha) + accum_b
            
            accum_alpha = alpha * (1.0 - accum_alpha) + accum_alpha
            
        # Advance ray
        pos_x += ray_increment[0]
        pos_y += ray_increment[1]
        pos_z += ray_increment[2]
        
        steps_taken += 1
        
    # 3. Finalization
    # Cap colors
    accum_r = min(accum_r, 1.0)
    accum_g = min(accum_g, 1.0)
    accum_b = min(accum_b, 1.0)
    
    # For XRay, normalize by global max (implied by "normalization" note, though not explicitly in loop)
    if current_render_mode == 1 and global_max_val > 0:
        accum_r /= global_max_val
        accum_g /= global_max_val
        accum_b /= global_max_val
        accum_alpha = 1.0 # XRay typically opaque? Source doesn't specify alpha for XRay output, but sets accum_alpha=0 initially. 
                           # Standard XRay is often just color. We leave alpha as accumulated or set to 1.0 if required by caller.
                           # Source sets accum_alpha=0 initially and never updates it in XRay mode.
                           
    dynamic_info.Color[0] = accum_r
    dynamic_info.Color[1] = accum_g
    dynamic_info.Color[2] = accum_b
    dynamic_info.Color[3] = accum_alpha
    dynamic_info.NumberOfStepsTaken = steps_taken
```

**Parameters:**
*   `currentRenderMode`: Integer constant. `0` for MIP, `1` for XRay, `2` for Full. Default is `2` (Full) [S1, unofficial].
*   `globalMaximumScalarValue`: Static float used to normalize XRay mode output. Initialized to `0.0`, updated during ray casting to the maximum scalar value encountered in the dataset [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Data Type Support**: Only `VTK_UNSIGNED_CHAR` and `VTK_UNSIGNED_SHORT` are supported. Other types trigger a warning and do not render [S1, unofficial].
*   **Early Termination**: The loop breaks if `accum_opacity >= 1.0`. This is a standard optimization but must be handled correctly in MIP mode where opacity might not accumulate traditionally [S1, unofficial].
*   **XRay Normalization**: The source mentions `globalMaximumScalarValue` is used to normalize XRay output, but the pseudocode in the source does not explicitly show the division step inside the loop or at the end, only that it is initialized and updated. The implementer must ensure normalization occurs to keep values in [0,1] [S1, unofficial].
*   **Shading Tables**: The source mentions shading tables but the pseudocode simplifies shading to `opacity * color`. In a full implementation, the shading tables (diffuse/specular) must be applied to the raw color values before accumulation in Full mode [S1, unofficial].

## Subtopics

### [Standard Ray Casting](12643-standard-ray-casting.md)
The base method using front-to-back compositing without multiple modes.

### [Maximum Intensity Projection](12643-standard-ray-casting.md)
A sibling technique often used in MIP mode, focusing on the highest intensity along the ray.

### [Shading and Lighting Techniques](8281-lighting-and-shading-techniques.md)
Techniques used in Full mode to enhance depth perception.

## Operating characteristics
*   **Time Complexity**: $O(N)$ per ray, where $N$ is the number of steps. Early termination can reduce this in opaque regions.
*   **Memory Complexity**: $O(1)$ per ray, as it only stores accumulators and current position. The volume data is accessed directly.
*   **Performance**: Depends heavily on the volume size and transfer function evaluation cost. Nearest-neighbor interpolation is faster than trilinear but may produce aliasing artifacts.
*   **Strengths**: Supports multiple rendering modes in a single pass; explicit shading control; early termination.
*   **Limitations**: Limited to integer data types; nearest-neighbor interpolation may cause jagged edges; XRay normalization requires global data pass.

## Background
This method is a custom implementation of volume ray casting, originally developed as part of the VTK (Visualization Toolkit) examples. It extends the `vtkVolumeRayCastCompositeFunction` to provide additional flexibility in rendering modes and shading. It is documented in unofficial VTK examples and tutorials [S1, unofficial].

## Sources
[S1] TotteKarlsson/vtkExamples, https://github.com/TotteKarlsson/vtkExamples, fetched 2026-10-27 (unofficial)
