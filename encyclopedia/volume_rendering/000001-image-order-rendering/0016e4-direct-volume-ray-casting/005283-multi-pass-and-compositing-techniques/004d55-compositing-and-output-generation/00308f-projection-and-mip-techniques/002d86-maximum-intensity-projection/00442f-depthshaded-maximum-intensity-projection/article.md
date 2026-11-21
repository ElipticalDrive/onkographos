# DepthShaded Maximum Intensity Projection

Also called: DEMIP with Depth Weighting, DSMIP, Depth-enhanced Maximum Intensity Projection (DeMIP)

## Overview
DepthShaded Maximum Intensity Projection (DSMIP) is a volume rendering technique that enhances depth perception in Maximum Intensity Projection (MIP) by modulating the final color based on the depth of the sampled structures [S1, unofficial][S3]. While standard MIP highlights high-intensity structures, it lacks depth cues, leading to ambiguities in interpreting spatial arrangement [S2, unofficial]. DSMIP addresses this by blending the standard MIP color with a depth-derived color, allowing viewers to distinguish between foreground and background structures with minimal information loss [S1, unofficial][S3]. It differs from [Local Maximum Intensity Projection](6054-local-maximum-intensity-projection.md) by retaining the global maximum along the ray but altering its visual presentation, and from [Fast Maximum Intensity Projection (MIP)](6635-fast-maximum-intensity-projection-mip.md) by adding a secondary depth-evaluation step [S3].

## How it works
**Inputs:**
*   Volume dataset $V$.
*   Camera parameters (view direction, position).
*   Depth weight parameter $d_w$ (typical value: 0.15 or 0.3) [S3].

**Outputs:**
*   Final color image $C_{final}$.

**Data Structures:**
*   `MIP_color`: The color associated with the maximum intensity voxel found along the ray.
*   `depth`: The normalized depth value ($0 \le depth \le 1$) of the fragment/voxel contributing to the MIP result, where 0 is the closest to the viewer and 1 is the farthest [S3].

**Steps:**
1.  **Primary Ray Casting (MIP Pass):** Cast a ray through the volume. Find the voxel with the maximum intensity value along the ray. Record its color as `MIP_color` and its depth as `depth` [S2, unofficial][S3].
2.  **Depth Shading Calculation:** Compute the final color by blending the `MIP_color` with a depth-derived intensity. The formula is:
    $$ C_{final} = MIP_{color} \times (1 - d_w) + 2 \times d_w \times (1 - depth) $$
    [S3]
    *   If $d_w = 0$, the result is the standard MIP color [S3].
    *   The term $(1 - depth)$ maps the depth to color intensity, making closer structures brighter in the depth component [S3].
3.  **Alternative Implementation (Modulation):** In some descriptions, the intensity of sampled values is modulated by their depth, and the result is written to the final image only if the modulated value is greater than the currently processed pixel's value [S2, unofficial]. This approach effectively suppresses far-away high-intensity values that might obscure closer features [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def depth_shaded_mip(volume, camera, depth_weight):
    """
    Renders a single frame using DepthShaded MIP.
    
    Args:
        volume: 3D array of voxel intensities.
        camera: Object containing ray generation logic.
        depth_weight: Float d_w (e.g., 0.15).
        
    Returns:
        image: 2D array of RGB colors.
    """
    width, height = camera.resolution
    image = zeros((height, width, 3))
    
    for y in range(height):
        for x in range(width):
            ray = camera.generate_ray(x, y)
            
            # Step 1: Find global maximum along the ray
            max_intensity = -infinity
            mip_color = black
            depth_at_max = 0.0
            
            # Iterate through samples along the ray
            for t in range(0, ray.length, ray.step):
                pos = ray.origin + t * ray.direction
                intensity = sample_volume(volume, pos)
                
                # Standard MIP logic: keep the highest intensity
                if intensity > max_intensity:
                    max_intensity = intensity
                    mip_color = get_color_from_intensity(intensity)
                    depth_at_max = t / ray.length # Normalize depth
            
            # Step 2: Apply depth shading formula
            # Formula: color = MIPcolor * (1 - dw) + 2 * dw * (1 - depth)
            # Note: The second term is a scalar intensity that modulates the color
            # or adds a tint. Based on [S3], it appears to be a scalar added to
            # the color components or used as a luminance boost.
            # Assuming mip_color is RGB and the second term is a scalar intensity
            # that is applied to the color (e.g., multiplying or adding).
            # The source formula implies the second term is a scalar value.
            # To keep it valid Python, we assume the second term is added to
            # the color channels or used as a multiplier if interpreted as luminance.
            # However, the formula explicitly writes:
            # color = MIPcolor * (1 - dw) + 2 * dw * (1 - depth)
            # This implies the second term is added to the RGB vector.
            # Since (1-depth) is scalar, we must broadcast it or assume it applies
            # to a grayscale version or the source implies a specific color mapping.
            # Given "MIPcolor" is likely RGB, and the second term is scalar,
            # we interpret this as adding a depth-based intensity to the color.
            
            depth_factor = 1.0 - depth_at_max
            depth_term = 2.0 * depth_weight * depth_factor
            
            # Apply the blend
            # If mip_color is normalized [0,1], and depth_term is intensity [0,1]
            # We add the depth term to the color.
            final_color = mip_color * (1.0 - depth_weight) + depth_term
            
            image[y, x] = final_color
            
    return image
```
[G] The pseudocode assumes `depth_at_max` is normalized to $[0,1]$. The source formula adds a scalar to an RGB vector; in implementation, this often means the scalar is broadcast across RGB channels or represents a luminance addition. The exact color mapping (e.g., spherical texture map mentioned in [S3] for other variants) is not detailed in the core formula, so the scalar addition is used as the direct translation of Eq. 1.

## Subtopics

### Depth Perception in MIP via Polygonal Surfaces
A method to improve depth perception in MIP by extracting isosurfaces and modifying vertex coordinates before projection [S1, unofficial]. It replaces the z-coordinate of each vertex with its associated isovalue, modifying the depth buffer so that geometry with higher densities appears closer to the viewpoint [S1, unofficial]. This modified z-buffer is then used to enhance the MIP visualization [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Slower than standard MIP due to the extra ray-casting step or depth evaluation [S3]. Faster than Direct Volume Rendering (DVR) because the per-sample computation (shading) is simpler than the full DVR transfer function and lighting model [S3].
*   **Measured Performance:** On an Intel Core 2 Duo E8400 and NVIDIA GeForce GTX 280, DSMIP achieved 53.8 fps for a 512x512x40 Jaw model, compared to 64.7 fps for standard MIP [S3]. For a 512x512x485 Head model, DSMIP achieved 25.8 fps vs 28.9 fps for MIP [S3].
*   **Strengths:** Enhances depth perception with minimal information loss; provides color cues for spatial context [S1, unofficial][S3].
*   **Limitations:** Values located far away have less chance of being projected if using the modulation variant, potentially obscuring volumetric features even if they have high intensity [S2, unofficial]. Artifacts may appear near bounding box boundaries when rotating the object if not implemented carefully [S3].
*   **When to use:** When depth cues are critical for interpreting the spatial arrangement of high-intensity structures (e.g., vascular vessels) and performance constraints allow for slightly more computation than standard MIP [S1, unofficial][S3].

## Background
The method was introduced to address the lack of depth cues in standard MIP, which can lead to ambiguities in interpreting the resulting image [S2, unofficial]. It is based on slightly modifying the MIP shading to take into account the depth of closer structures with similar material [S1, unofficial]. The technique is also known as Depth-enhanced Maximum Intensity Projection (DeMIP) [S1, unofficial][S3]. It was developed as an alternative to polygon-based depth enhancement methods [S1, unofficial].

## Sources
[S1] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
[S2] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
[S3] Depth-Enhanced Maximum Intensity Projection, https://doi.org/10.2312/vg/vg10/093-100, fetched 2026-10-29
