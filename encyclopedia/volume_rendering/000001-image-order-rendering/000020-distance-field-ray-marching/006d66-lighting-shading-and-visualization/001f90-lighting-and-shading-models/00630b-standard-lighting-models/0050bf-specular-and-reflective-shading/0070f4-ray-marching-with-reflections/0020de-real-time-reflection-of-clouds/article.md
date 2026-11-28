# Real-time Reflection of Clouds

Also called: Panoramic Cloud Reflection, SSR for Clouds, Sky Light Cloud Reflections, Volumetric Cloud Reflection Tracing

## Overview
This method provides scene reflections for volumetric clouds by ray marching reflection surfaces using data from the Sky Light component [S2]. It addresses the limitation of standard Screen-Space Reflections (SSR), which can only reflect clouds currently visible within the screen space [S1, unofficial]. To capture clouds outside the current view, it renders a low-resolution panoramic view in an extra pass and blends it into the sky cubemap [S1, unofficial]. This approach is distinct from [Screen-Space Reflections](9698-screen-space-reflections.md) by its reliance on a pre-captured panoramic texture rather than real-time depth buffers, and from [Ray Marching with Refraction and Beer's Law Absorption](1947-ray-marching-with-refraction-and-beer-s-law-absorption.md) by its specific focus on reflective sky lighting rather than transmissive volume scattering.

## How it works
**Inputs:**
*   Volumetric Cloud component data (density, height, position).
*   Sky Light component (environmental lighting, cubemap).
*   Camera view parameters.

**Outputs:**
*   Reflection contribution to the final cloud shading.

**Data Structures:**
*   **Panoramic Cloud Texture:** A low-resolution texture, typically $256 \times 128$ pixels [S1, unofficial].
*   **Sky Cubemap:** The environment map used for lighting, updated with the panoramic cloud data [S1, unofficial].
*   **Reflection Sample Count:** A scalar value scaled by `Reflection Sample Count Scale` [S2].
*   **Shadow Reflection Sample Count:** A scalar value scaled by `Shadow Reflection Sample Count Scale` [S2].

**Parameters:**
*   **Reflection Sample Count Scale:** Slider in the Volumetric Cloud Details panel under Cloud Tracing [S2].
    *   Progressive values: 0.25, 0.5, 0.75, 1 (Default), 2, 4, 8 [S2].
    *   Clamped by default [S2].
    *   Max count controlled by console command `r.VolumetricCloud.ReflectionRaySampleMaxCount` [S2].
*   **Shadow Reflection Sample Count Scale:** Slider in the Volumetric Cloud Details panel under Cloud Tracing [S2].
    *   Progressive values: 0.25, 0.5, 0.75, 1 (Default), 2, 4, 8 [S2].
    *   Clamped by default [S2].
    *   Max count controlled by console command `r.VolumetricCloud.Shadow.ReflectionRaySampleMaxCount` [S2].

**Steps:**
1.  **Render Panoramic Cloud View:** In a separate pass, render a low-resolution ($256 \times 128$) panoramic view of the clouds [S1, unofficial].
2.  **Blend into Sky Cubemap:** Blend the rendered panoramic cloud data into the Sky Light's cubemap [S1, unofficial].
3.  **Ray March Reflections:** During the main cloud shading pass, use the Sky Light component to ray march reflection surfaces in the scene [S2].
4.  **Sample Count Application:** Scale the number of samples used for tracing the cloud's reflection ray based on the `Reflection Sample Count Scale` [S2].
5.  **Shadowed Reflections:** Apply `Shadow Reflection Sample Count Scale` to determine the sample count for shadowed reflections of clouds [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_cloud_reflection(cloud_data, sky_light, reflection_scale, shadow_reflection_scale):
    # Step 1 & 2: Ensure panoramic cloud is blended into sky cubemap
    # This is typically done in a separate render pass, not per-pixel.
    # sky_light.cubemap = blend(sky_light.cubemap, render_low_res_panoramic_cloud(cloud_data))
    
    # Step 3: Determine base sample count from Sky Light
    # The exact base count is implementation-specific, but scaled by the provided factors.
    base_samples = get_base_ray_march_samples(sky_light)
    
    # Step 4: Calculate reflection samples
    reflection_samples = base_samples * reflection_scale
    
    # Step 5: Calculate shadow reflection samples
    shadow_reflection_samples = base_samples * shadow_reflection_scale
    
    # Ray march reflection surfaces using the calculated sample counts
    # The ray direction is determined by the reflection vector of the view ray
    # against the cloud surface normal, sampled from the Sky Light cubemap.
    reflected_color = ray_march_reflection(
        origin=cloud_data.position,
        direction=reflect(view_direction, cloud_data.normal),
        sample_count=reflection_samples,
        environment_map=sky_light.cubemap
    )
    
    shadowed_reflected_color = ray_march_reflection(
        origin=cloud_data.position,
        direction=reflect(view_direction, cloud_data.normal),
        sample_count=shadow_reflection_samples,
        environment_map=sky_light.cubemap,
        shadowed=True
    )
    
    return reflected_color, shadowed_reflected_color
```

**Edge Cases and Pitfalls:**
*   **Screen Space Limitation:** Standard SSR only reflects clouds within screen space [S1, unofficial]. This method extends that by using the panoramic pass, but the panoramic texture itself has low precision [S1, unofficial].
*   **Precision:** The method has low precision and is suitable only for blurry reflections [S1, unofficial].
*   **Water Reflections:** For water surfaces, this method is insufficient; separate specular handling is required for cloud reflections on water [S1, unofficial].

## Subtopics

### Sky Light Cloud Reflections Quality
Controls the sample counts for cloud reflections via the Volumetric Cloud Details panel [S2].

### Ray Marching Quality Modes
Scalable modes (reactive to cinematic) that affect how cloud intersections with opaque surfaces are handled, influencing the underlying ray marching performance [S2].

## Operating characteristics
*   **Time Complexity:** Dependent on the `Reflection Sample Count Scale` and `Shadow Reflection Sample Count Scale`. The base cost includes an extra render pass for the $256 \times 128$ panoramic cloud view [S1, unofficial].
*   **Memory Complexity:** Requires storage for the low-resolution panoramic texture and the updated Sky Cubemap [S1, unofficial].
*   **Performance:** The low resolution ($256 \times 128$) of the panoramic pass keeps the overhead manageable for real-time applications [S1, unofficial]. However, the low precision limits its use to blurry reflections [S1, unofficial].
*   **Strengths:** Captures reflections of clouds outside the current screen space, which SSR cannot do [S1, unofficial].
*   **Limitations:** Low precision; not suitable for sharp reflections or water surfaces [S1, unofficial].
*   **When to use:** When blurry, screen-space-independent cloud reflections are needed in real-time volumetric cloud rendering.
*   **When not to use:** When sharp reflections are required, or for water surface reflections [S1, unofficial].

## Background
This method is part of the volumetric cloud rendering pipeline in Unreal Engine, specifically leveraging the Sky Light component for environmental reflections [S2]. It addresses the artifacts and limitations of pure screen-space approaches by incorporating a panoramic capture pass [S1, unofficial]. The implementation details, such as the sample count scales and console commands, are documented in the Unreal Engine 5.8 documentation for the Volumetric Cloud Component [S2]. The conceptual basis for handling reflections in atmospheric scattering and cloud rendering, including the use of transmittance LUTs and depth blending, is discussed in general volumetric cloud implementation guides [S1, unofficial].

## Sources
[S1] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
[S2] Volumetric Cloud Component in Unreal Engine | Unreal Engine 5.8 Documentation | Epic Developer Community, https://dev.epicgames.com/documentation/en-us/unreal-engine/volumetric-cloud-component-in-unreal-engine, fetched 2026-10-30
