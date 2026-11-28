# Segmented Dataset Rendering with Anatomical Feature Enhancement

Also called: Multi-Label Volume Rendering, Segmentation-Based Volume Rendering, Segmentation-Guided Ray Casting, Sliced Volume Rendering with Segmentation Labels

## Overview
Segmented Dataset Rendering with Anatomical Feature Enhancement is a ray casting technique that replaces scalar-based transfer functions with discrete segmentation labels to assign pre-defined colors and visibility flags per anatomical structure [S1, unofficial][S2]. It solves the problem of jagged subvolume boundaries and inconsistent shading in categorical volume rendering by "painting" colors acquired from segmentation directly, delivering higher image quality with smoother subvolume boundaries compared to using different transfer functions for segmented object separation [S2]. Unlike its parent [iso_categorical](4049-iso-categorical.md), which marches along view rays in fixed steps and tests only the voxel at each sample point (causing sawtooth edges at grazing angles), this method uses the segmentation texture to determine material identity and applies per-label clipping and color mapping, allowing for distinct rendering modes per object [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A segmentation texture (volume) containing integer labels, and a set of uniform parameters defining colors, visibility, and clipping for each label [S1, unofficial].
*   **Output:** A 2D image with front-to-back compositing [S1, unofficial].

**Data Structures:**
The method relies on three uniform arrays of size 25, corresponding to the maximum number of supported segmentation labels (0-24) [S1, unofficial].
1.  `colors[25]`: An array of 25 `vec3` uniforms defining the RGB color for each of the 25 possible segmentation labels [S1, unofficial].
2.  `displays[25]`: An array of 25 `bool` uniforms indicating whether each segmentation label is visible [S1, unofficial].
3.  `clips[25]`: An array of 25 `bool` uniforms indicating whether each segmentation label is subject to clipping by a bounding box [S1, unofficial].
4.  `box_min` and `box_max`: `vec3` uniforms defining the axis-aligned bounding box for clipping [S1, unofficial].
5.  `use_seg`: A boolean uniform that toggles between segmentation-based rendering and standard transfer-function-based rendering [S1, unofficial].

**Numbered Steps:**
1.  For each pixel, cast a ray through the volume [S1, unofficial].
2.  At each sample point `p` along the ray:
    a. Sample the segmentation texture at position `p` to retrieve the label value `seg`. The source specifies the extraction as `seg = int(255.0 * texture(segmentation, p).r)` [S1, unofficial]. This implies the segmentation data is stored in the red channel of a floating-point texture, normalized to [0, 1], where the integer label is derived by scaling by 255 and truncating [S1, unofficial].
    b. Determine if the current segment `seg` should be displayed by looking up the `displays` array: `use = displays[seg]` [S1, unofficial].
    c. If `use` is true, determine if the point should be clipped based on the `clips` array and the bounding box `box_min`/`box_max`. If `clips[seg]` is true, the point is only rendered if it is inside the box (i.e., `box_min <= p <= box_max`) [S1, unofficial]. If the point is outside the box and clipping is enabled, skip this sample [S1, unofficial].
    d. If the point is rendered, set the color using the pre-defined `colors` array for the segment. The alpha value is derived from the volume value `val` at that position: `val_color = vec4(colors[seg], val)` [S1, unofficial]. The source does not specify if `val` comes from a separate scalar volume or the segmentation intensity itself; a scalar volume for opacity is typically used alongside the segmentation texture in anatomical feature enhancement contexts [G].
    e. Apply opacity correction and front-to-back compositing as in the base method [S1, unofficial].
3.  If `use_seg` is false, fall back to standard transfer-function-based rendering [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_segmented_volume(ray_origin, ray_direction, volume_dimensions, segmentation_texture, scalar_texture, 
                            colors, displays, clips, box_min, box_max, use_seg, step_size):
    """
    Renders a volume using segmentation-based rendering with anatomical feature enhancement.
    
    Args:
        ray_origin: vec3, origin of the view ray.
        ray_direction: vec3, direction of the view ray.
        volume_dimensions: vec3, dimensions of the volume in voxels.
        segmentation_texture: function, samples the segmentation texture at normalized coordinates (0-1).
        scalar_texture: function, samples the scalar texture at normalized coordinates (0-1).
        colors: list of vec3, size 25, RGB colors for each label.
        displays: list of bool, size 25, visibility flags for each label.
        clips: list of bool, size 25, clipping flags for each label.
        box_min: vec3, minimum corner of the clipping bounding box.
        box_max: vec3, maximum corner of the clipping bounding box.
        use_seg: bool, if True, use segmentation-based rendering; otherwise, use standard TF.
        step_size: float, step size along the ray in voxel units.
        
    Returns:
        vec4, the final color of the pixel.
    """
    if not use_seg:
        # [G] Standard transfer-function-based rendering fallback.
        # Implementation details not provided in sources.
        return vec4(0.0, 0.0, 0.0, 0.0)

    accumulated_color = vec4(0.0, 0.0, 0.0, 0.0)
    accumulated_alpha = 0.0
    
    # Compute max t to cover the volume bounding box [0,0,0] to volume_dimensions
    max_t = 0.0
    for i in range(3):
        if ray_direction[i] != 0.0:
            t_far = volume_dimensions[i] / ray_direction[i]
            if t_far > max_t:
                max_t = t_far
        else:
            max_t = max(max_t, volume_dimensions[i])
            
    t = 0.0
    while t < max_t:
        p = ray_origin + ray_direction * t
        
        # Check if point is within volume bounds [0, volume_dimensions)
        if not (0.0 <= p.x < volume_dimensions.x and 
                0.0 <= p.y < volume_dimensions.y and 
                0.0 <= p.z < volume_dimensions.z):
            t += step_size
            continue
            
        # Sample segmentation texture
        seg_float = segmentation_texture(p)
        seg = int(255.0 * seg_float.r)
        
        # Determine if segment should be displayed
        use = displays[seg]
        
        if use:
            # Check clipping
            if clips[seg]:
                if not (box_min.x <= p.x <= box_max.x and 
                        box_min.y <= p.y <= box_max.y and 
                        box_min.z <= p.z <= box_max.z):
                    t += step_size
                    continue
            
            # Get color for the segment
            color_rgb = colors[seg]
            
            # Get alpha from volume value
            val = scalar_texture(p)
            alpha = val
            
            # Front-to-back compositing
            contribution = vec4(color_rgb, alpha)
            accumulated_color = accumulated_color + (1.0 - accumulated_alpha) * contribution
            accumulated_alpha = accumulated_alpha + (1.0 - accumulated_alpha) * alpha
            
            if accumulated_alpha >= 1.0:
                break
        
        t += step_size

    return accumulated_color
```

**Parameters with Typical Values:**
*   `colors[25]`: 25 `vec3` uniforms. Values depend on the specific anatomical structures being rendered [S1, unofficial].
*   `displays[25]`: 25 `bool` uniforms. Typically set to `true` for structures of interest [S1, unofficial].
*   `clips[25]`: 25 `bool` uniforms. Typically set to `true` for structures that should be confined to a specific region [S1, unofficial].
*   `use_seg`: Boolean. Set to `true` to enable segmentation-based rendering [S1, unofficial].
*   `box_min`, `box_max`: `vec3` uniforms. Define the bounding box for clipping [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The segmentation label extraction uses `int(255.0 * texture(segmentation, p).r)`. This assumes the segmentation labels are stored as floating-point values in the range [0, 1] where the integer label `L` is represented as `L/255.0` [S1, unofficial].
*   **Boundary Conditions:** The clipping check uses inclusive bounds (`<=`). Points exactly on the boundary are considered inside [S1, unofficial].
*   **Missing Scalar Volume:** The method uses `val` for alpha. If no scalar volume is available, the behavior is undefined in the sources [G]. It is recommended to use a default alpha of 1.0 or the segmentation intensity if no scalar volume is present.

## Subtopics

### [Two-level volume rendering](7677-two-level-volume-rendering.md)
Two-level volume rendering (2lVR) is an extension of standard 3D visualization that enables object-aware volume rendering by subdividing a volume into distinct "objects" (structures of interest) and assigning individual rendering modes to each. It solves the problem of rendering segmented datasets with high image quality and smooth subvolume boundaries by allowing different transfer functions, shading models, or compositing strategies per object. Unlike standard direct volume rendering which applies a uniform model to the entire dataset, 2lVR distinguishes between volume objects using explicit identity numbers, allowing separate shading and compositing properties for each.

### Mesh-Based Classification for Volume Rendering
A volume rendering method starting from an MRI scan and segmented organ meshes, utilizing known meshes to define the transfer function at each raycasting sample to resolve intersection ambiguities and remove staircasing artifacts [S3, unofficial]. It renders animated sequences of MRIs using a CPU volume renderer written in C++, built on Ubuntu 20.04 Linux, Intel Core i7-7700K processor with 8 cores and 40 GB RAM [S3, unofficial]. It uses Intel Embree library (v3.4.0) for ray-object intersections and oneTBB (v2021.5.1) for parallel pixel computation [S3, unofficial]. Input MRI volume data resolution is 400x400x400 with voxel size 0.64mm x 0.64mm x 0.64mm [S3, unofficial]. Average rendering time for 1024x1024 resolution image is 3.7 seconds for fat-emphasized style and 4.7 seconds for interior-emphasized style, with maximum memory usage of 612.3 MB [S3, unofficial]. Compared to surface rendering, this method shows outer surface and internal structures simultaneously without transparency, solves penetration artifacts between structures (e.g., ligaments/bones) via priority assignment, and allows linear combination of materials [S3, unofficial]. Compared to existing quality methods, it provides clearer boundaries between tissues in both interior-emphasized and fat-emphasized styles [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples along each ray, similar to standard ray casting [G]. The additional cost is the texture lookup for segmentation and the conditional checks for visibility and clipping, which are negligible compared to the ray marching loop [G].
*   **Memory Complexity:** Requires storage for the segmentation texture, the scalar texture (if used), and the uniform arrays (`colors`, `displays`, `clips`) [S1, unofficial]. The uniform arrays are small (size 25) [S1, unofficial].
*   **Measured Performance:** Not given in the sources for the base method [S1, unofficial][S2].
*   **Scaling:** Scales with the resolution of the segmentation texture and the number of samples per ray [G].
*   **Strengths:** Provides smoother subvolume boundaries compared to standard transfer-function-based segmentation [S2]. Allows for explicit control over visibility and clipping per label [S1, unofficial].
*   **Limitations:** Limited to 25 labels due to the fixed-size uniform arrays [S1, unofficial]. Requires a segmentation texture as input, which may not always be available [S1, unofficial]. The clipping is axis-aligned and box-based [S1, unofficial].
*   **When to Use:** When rendering segmented datasets with a small number of distinct anatomical structures and when smooth boundaries and explicit control over per-label rendering are desired [S2].
*   **When Not to Use:** Not given in the sources.

## Background
The method was presented as a new algorithm for rendering segmented datasets with anatomical feature enhancement, following the development of techniques for dynamic volume rendering and manipulation [S2]. It was introduced to address the limitations of using different transfer functions for segmented object separation, which could result in lower image quality and jagged boundaries [S2]. The technique is part of a broader effort to facilitate image manipulation and image-guided intervention, including the development of "depth texture indexing" for virtual surgical tools [S2].

## Sources
[S1] johanna-b/VisWeb, https://github.com/johanna-b/VisWeb, fetched 2026-10-27 (unofficial)
[S2] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S3] Volume Rendering of Human Hand Anatomy, https://arxiv.org/abs/2411.18630v1, fetched 2026-10-26 (unofficial)
