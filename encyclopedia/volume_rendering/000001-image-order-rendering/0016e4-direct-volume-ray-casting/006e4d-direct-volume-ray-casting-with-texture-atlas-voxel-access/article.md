# Direct Volume Ray Casting with Texture Atlas Voxel Access

Also called: 2D Texture Atlas Ray Casting, Voxelization via 2D Texture

## Overview
Direct Volume Ray Casting with Texture Atlas Voxel Access is a hardware-accelerated technique that renders 3D volumetric data by packing discrete slices into a 2D texture atlas and accessing them via bilinear interpolation in a fragment shader [S1]. It differs from standard direct volume ray casting by replacing 3D texture lookups with 2D texture lookups, allowing the use of standard 2D texture units and hardware filtering on GPUs that may lack efficient 3D texture support or where memory bandwidth for 3D textures is a bottleneck [S1]. It differs from [Discrete Ray Casting with Trilinear Reconstruction](21585-discrete-ray-casting-with-trilinear-reconstruction.md) by relying on pre-sliced 2D tiles and bilinear interpolation within those tiles, rather than reconstructing continuous values via trilinear interpolation across the full 3D volume space [S1].

## How it works
The method casts rays through a volume represented as a 2D texture atlas. The volume is sliced into `numberOfSlices` layers, packed into a 2D grid of `slicesOverX` by `slicesOverY` tiles, and accessed via bilinear interpolation in the fragment shader [S1].

### Inputs and Outputs
**Inputs:**
*   **Volume Data:** A 3D volume of size $W \times H \times D$, where $D$ is the number of slices (`numberOfSlices`). The data is pre-processed into a 2D texture atlas.
*   **Camera:** Model-view matrix (`modelViewInv`) to compute ray origin and direction.
*   **Parameters:**
    *   `numberOfSlices`: Total number of volume slices along the Z-axis. Typical value: 96 [S1].
    *   `slicesOverX`: Number of slices along the X-axis of the 2D texture atlas grid. Typical value: 10 [S1].
    *   `slicesOverY`: Number of slices along the Y-axis of the 2D texture atlas grid. Typical value: 10 [S1].
    *   `numSamples`: Number of ray marching steps per ray. Typical value: 256 [S1].
    *   `opacityFactor`: Global opacity multiplier.
    *   `colorizeIntensity`: Boolean flag to enable heatmap coloring.
    *   `intensityMin`, `intensityMax`: Bounds for intensity isolation mask.
    *   `clipPlaneNormal`: Normal vector of the clipping plane.
    *   `clipPlaneDepth`: Depth of the clipping plane.
    *   `clip`: Boolean flag to enable clipping.
*   **Texture:** The 2D texture atlas containing the sliced volume data.

**Outputs:**
*   **Framebuffer:** RGBA8 color buffer, RED_INTEGER buffer (optional), and Depth buffer. Typical resolution: 1280x720 [S1].

### Data Structures
*   **2D Texture Atlas:** A 2D texture where each tile represents a slice of the volume. The atlas dimensions are determined by `slicesOverX` and `slicesOverY`. The total number of tiles is `slicesOverX * slicesOverY`. Each tile contains $W \times H$ texels.
*   **Ray State:**
    *   `rayStart`: 3D vector, ray origin in world space.
    *   `rayStop`: 3D vector, ray exit point in world space.
    *   `dir`: 3D vector, normalized ray direction.
    *   `pos`: 3D vector, current sample position.
    *   `marchDist`: 3D vector, step vector for ray marching.
    *   `travel`: float, remaining distance to march.
    *   `stepSize`: float, distance between samples.
*   **Accumulation Buffer:** `accum` (vec4), initialized to $(0, 0, 0, 0)$ [S1].

### Numbered Steps

1.  **Ray Definition:** Calculate ray origin and direction from the inverse model-view matrix. The ray origin is `modelViewInv[3].xyz`. The direction is normalized `pos - modelViewInv[3].xyz`, where `pos` is the vertex position of the cube mesh covering the view frustum [S1].
2.  **Ray-Volume Intersection:** Intersect the ray with the axis-aligned bounding box (AABB) of the volume, defined as $[-1, 1]^3$. Find the entry parameter `tnear` and exit parameter `tfar`. If `tnear < 0`, set `tnear = 0` to handle cases where the camera is inside the volume [S1].
3.  **Coordinate Transformation:** Transform the ray start and stop points from world space $[-1, 1]$ to texture space $[0, 1]$ by adding 1 and multiplying by 0.5 [S1].
4.  **Step Size Calculation:** Calculate the step size as `maxDist / numSamples`, where `maxDist` is the diagonal of the unit cube, $\sqrt{2}$ [S1].
5.  **Clipping (Optional):** If clipping is enabled, calculate the intersection distance `dis` with the clipping plane. Adjust `rayStart` or `rayStop` based on the front/back face orientation and `dis` bounds. Recalculate `marchDist`, `travel`, and `pos`. Handle the edge case where `dis == 0` to avoid division by zero [S1].
6.  **Initialization:** Initialize the accumulation buffer `accum` to zero. Calculate the initial `travel` distance and `marchDist` vector [S1].
7.  **Ray Marching Loop:** Loop `numSamples` times or until `travel <= 0`. In each iteration:
    a.  **Voxel Access:** Sample the voxel value at the current position `pos` using `getVoxel(tex, pos)`. This function maps the 3D volume coordinates to 2D texture coordinates based on the slice index and grid position [S1].
    b.  **Color Computation:** If `colorizeIntensity` is true, apply a heatmap transfer function `Heatmap(tf_pos, 0., 1.)` to the sampled value `tf_pos`. Otherwise, use `tf_pos` directly as the RGB color [S1].
    c.  **Alpha Masking:** Compute an alpha mask for intensity isolation using `smoothstep` with `intensityMin` and `intensityMax`. The formula is:
        $$
        \text{value.a} = \left( \text{smoothstep}(\text{intensityMax}, \frac{\text{intensityMax}+\text{intensityMin}}{2}, \text{tf\_pos}) \times \text{smoothstep}(\text{intensityMin}, \frac{\text{intensityMax}+\text{intensityMin}}{2}, \text{tf\_pos}) \right)^{0.25}
        $$
        [S1].
    d.  **Sample Alpha:** Compute the sample alpha:
        $$
        \text{vsample.a} = \text{value.a} \cdot \text{opacityFactor} \cdot \frac{1}{\text{numSamples}} \cdot \text{fade}
        $$
        where $\text{fade} = \text{clamp}\left(\frac{\text{distance}(\text{pos}, \text{rayStop})}{\text{stepSize}}, 0., 1.\right)$ [S1].
    e.  **Sample Color:** Compute the sample color:
        $$
        \text{vsample.rgb} = \text{value.rgb} \cdot \text{vsample.a} \cdot \text{lightFactor}
        $$
        where `lightFactor` is typically 1.5 [S1].
    f.  **Compositing:** Accumulate color and alpha using front-to-back blending:
        $$
        \text{accum.rgb} \leftarrow \text{accum.rgb} + (1. - \text{accum.a}) \cdot \text{vsample.rgb}
        $$
        $$
        \text{accum.a} \leftarrow \text{accum.a} + \text{vsample.a}
        $$
        Break the loop if `accum.a >= 1.` [S1].
8.  **Output:** Write `accum.rgb` and `accum.a` to the framebuffer [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_volume_atlas(tex, model_view_inv, num_samples, opacity_factor, colorize_intensity, intensity_min, intensity_max, clip_plane_normal, clip_plane_depth, clip_enabled):
    """
    Renders a volume using a 2D texture atlas.
    
    Args:
        tex: 2D texture atlas containing the volume slices.
        model_view_inv: 4x4 inverse model-view matrix.
        num_samples: Number of ray marching steps.
        opacity_factor: Global opacity multiplier.
        colorize_intensity: Boolean flag for heatmap coloring.
        intensity_min: Minimum intensity for isolation mask.
        intensity_max: Maximum intensity for isolation mask.
        clip_plane_normal: Normal vector of the clipping plane.
        clip_plane_depth: Depth of the clipping plane.
        clip_enabled: Boolean flag to enable clipping.
        
    Returns:
        accum: vec4, the accumulated color and alpha.
    """
    # Step 1: Ray Definition
    ray_origin = model_view_inv[3].xyz
    # Assuming 'pos' is the vertex position of the cube mesh covering the view frustum
    # In a real implementation, this would be per-vertex or per-pixel
    ray_direction = normalize(pos - ray_origin)
    
    # Step 2: Ray-Volume Intersection with AABB [-1, 1]^3
    # Simplified AABB intersection logic
    tnear = 0.0
    tfar = sqrt(2.0) # Diagonal of unit cube
    
    # Calculate step size
    step_size = tfar / num_samples
    
    # Step 3: Coordinate Transformation
    ray_start = (ray_origin + 1.0) * 0.5
    ray_stop = (ray_origin + ray_direction * tfar + 1.0) * 0.5
    
    # Step 4: Clipping (Optional)
    if clip_enabled:
        # Calculate intersection distance 'dis' with the clipping plane
        # Plane equation: dot(p, clip_plane_normal) + clip_plane_depth = 0
        # Ray: p = ray_start + t * ray_direction
        # dis = -(dot(ray_start, clip_plane_normal) + clip_plane_depth) / dot(ray_direction, clip_plane_normal)
        denominator = dot(ray_direction, clip_plane_normal)
        if abs(denominator) > 1e-6: # Avoid division by zero
            dis = -(dot(ray_start, clip_plane_normal) + clip_plane_depth) / denominator
            if dis > 0:
                if dis < tfar:
                    ray_stop = ray_start + ray_direction * dis
                    tfar = dis
        # Recalculate step size and travel distance
        step_size = tfar / num_samples
        travel = tfar
    else:
        travel = tfar
        
    march_dist = ray_direction * step_size
    pos = ray_start
    accum = vec4(0.0, 0.0, 0.0, 0.0)
    
    # Step 7: Ray Marching Loop
    for i in range(num_samples):
        if travel <= 0.0:
            break
            
        # a. Voxel Access
        tf_pos = get_voxel_atlas(tex, pos, num_slices=96, slices_over_x=10, slices_over_y=10)
        
        # b. Color Computation
        if colorize_intensity:
            value_rgb = heatmap(tf_pos, 0.0, 1.0)
        else:
            value_rgb = vec3(tf_pos)
            
        # c. Alpha Masking
        mid_intensity = (intensity_max + intensity_min) / 2.0
        smooth1 = smoothstep(intensity_max, mid_intensity, tf_pos)
        smooth2 = smoothstep(intensity_min, mid_intensity, tf_pos)
        value_a = pow(smooth1 * smooth2, 0.25)
        
        # d. Sample Alpha
        fade = clamp(distance(pos, ray_stop) / step_size, 0.0, 1.0)
        vsample_a = value_a * opacity_factor * (1.0 / float(num_samples)) * fade
        
        # e. Sample Color
        light_factor = 1.5
        vsample_rgb = value_rgb * vsample_a * light_factor
        
        # f. Compositing
        accum.rgb += (1.0 - accum.a) * vsample_rgb
        accum.a += vsample_a
        
        if accum.a >= 1.0:
            break
            
        # Update position
        pos += march_dist
        travel -= step_size
        
    return accum

def get_voxel_atlas(tex, pos, num_slices, slices_over_x, slices_over_y):
    """
    Maps 3D volume coordinates to 2D texture coordinates and samples the atlas.
    
    Args:
        tex: 2D texture atlas.
        pos: 3D position in texture space [0, 1].
        num_slices: Total number of volume slices.
        slices_over_x: Number of slices along X in the atlas.
        slices_over_y: Number of slices along Y in the atlas.
        
    Returns:
        float: The sampled voxel value.
    """
    # Invert X coordinate as per source
    volpos_x = 1.0 - pos.x
    volpos_y = pos.y
    volpos_z = pos.z
    
    # Calculate slice index
    s1 = floor(volpos_z * num_slices)
    
    # Calculate texture coordinates for the two adjacent slices
    # texpos1.x = dx1 + volpos.x / -slicesOverX
    # dx1 = fract(s1 / -slicesOverX)
    # Note: The source formula uses negative division, which might be a specific coordinate system detail.
    # Assuming standard positive grid layout for clarity, but adhering to source logic:
    
    dx1 = fract(s1 / -slices_over_x)
    texpos1_x = dx1 + volpos_x / -slices_over_x
    texpos1_y = floor(s1 / slices_over_y) / slices_over_y + volpos_y / slices_over_y
    
    # Calculate texture coordinates for the next slice
    s2 = s1 + 1
    dx2 = fract(s2 / -slices_over_x)
    texpos2_x = dx2 + volpos_x / -slices_over_x
    texpos2_y = floor(s2 / slices_over_y) / slices_over_y + volpos_y / slices_over_y
    
    # Sample the two slices
    value1 = textureLod(tex, vec2(texpos1_x, texpos1_y), 0).x
    value2 = textureLod(tex, vec2(texpos2_x, texpos2_y), 0).x
    
    # Interpolate between the two slices
    # tf_pos = mix(value1, value2, volpos.z * num_slices - s1)
    t = volpos_z * num_slices - s1
    tf_pos = mix(value1, value2, t)
    
    return tf_pos
```

### Edge Cases and Pitfalls
*   **Clipping Plane Division by Zero:** The intersection distance `dis` calculation involves division by the dot product of the ray direction and the clipping plane normal. If this dot product is zero, the ray is parallel to the plane. The code must check `if (dis != 0.)` or `if (abs(denominator) > epsilon)` before calculating `dis` [S1].
*   **Camera Inside Volume:** The ray start time `tnear` can be negative if the camera is inside the AABB. The code clamps `tnear` to 0 to prevent negative march distances [S1].
*   **Voxel Coordinate Inversion:** The voxel coordinate X is inverted in `getVoxel` via `volpos.x = 1.-volpos.x` before texture lookup. This must be accounted for in the coordinate mapping [S1].
*   **Precision:** The use of `fract` and `floor` operations for texture coordinate calculation can introduce precision errors, especially for large volumes. Using higher precision types or careful normalization can mitigate this [G].

## Subtopics

### [Discrete Ray Casting with Trilinear Reconstruction](21585-discrete-ray-casting-with-trilinear-reconstruction.md)
Discrete Ray Casting with Trilinear Reconstruction is a direct volume rendering technique that samples volumetric data using discrete ray casting steps and reconstructs scalar values via trilinear interpolation. It differs from its parent, Direct Volume Ray Casting with Texture Atlas Voxel Access, by employing trilinear reconstruction within the ray casting loop rather than relying solely on bilinear interpolation of pre-sliced texture tiles for access. This method allows for continuous sampling along the ray path through the volume, improving visual quality over nearest-neighbor or purely 2D-interpolated approaches while maintaining the hardware-accelerated texture access patterns of the atlas-based parent.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of samples per ray and $P$ is the number of pixels. The complexity is linear with respect to the number of samples and pixels.
*   **Memory Complexity:** $O(W \cdot H \cdot D)$ for the volume data, but stored as a 2D texture atlas, so the memory footprint is similar to a 2D texture of size $(W \cdot \text{slicesOverX}) \times (H \cdot \text{slicesOverY})$.
*   **Measured Performance:** The method is hardware-accelerated and runs in the fragment shader. Performance depends on the GPU's texture sampling throughput and the number of samples per ray. Typical resolution: 1280x720, 256 samples per ray [S1].
*   **Scaling:** The method scales well with resolution and sample count, but performance degrades linearly with increased samples. The 2D texture atlas approach can be more memory-efficient than 3D textures on some hardware.
*   **Strengths:** Hardware-accelerated, efficient memory access patterns, compatible with standard 2D texture units.
*   **Limitations:** Limited to discrete slices, potential artifacts at slice boundaries if not interpolated correctly, less flexible than 3D texture lookups for arbitrary sampling positions.
*   **When to Use:** When rendering large volumes on hardware with limited 3D texture support or when memory bandwidth is a concern.
*   **When Not to Use:** When high-quality continuous interpolation is required, or when the volume data changes frequently and repacking the atlas is expensive.

## Background
The method is based on the implementation in the MobiusEngine, specifically the `VolumeInspector` component [S1]. The technique of using texture atlases for volume rendering has been explored in various contexts to optimize memory access and leverage 2D texture hardware. The specific implementation details, including the clipping plane handling and intensity isolation mask, are derived from the `VolumeInspector` source code [S1].

## Sources
[S1] graphXDoses/VolumeInspector, https://github.com/graphXDoses/VolumeInspector, fetched 2026-10-28
