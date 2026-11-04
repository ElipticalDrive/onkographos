# Performance Optimizations for SDF Ray Marching

Also called: Optimisations, SDF Ray Marching Optimizations

## Overview
Performance Optimizations for SDF Ray Marching refers to a set of techniques designed to reduce the computational cost of rendering scenes defined by Signed Distance Fields (SDFs) while maintaining visual fidelity. It solves the primary bottleneck of ray marching: the high number of SDF evaluations and shader instructions required per pixel, particularly in complex or high-resolution scenes. What sets these optimizations apart from sibling methods like [Beam Optimization](932-beam-optimization.md) is their focus on reducing the *number* of march steps or the *cost* of each step through geometric culling, resolution scaling, and parameter tuning, rather than changing the fundamental traversal structure or parallelism model.

## How it works
The method consists of several independent optimization strategies applied during the ray marching loop or the rendering pipeline setup.

### Inputs and Outputs
*   **Inputs:** Scene SDF functions, camera parameters, ray origin/direction, maximum step count (`MAX_STEPS`), epsilon threshold (`EPSILON`), and resolution settings.
*   **Outputs:** Rendered image with reduced computational cost.

### Data Structures
No new data structures are strictly required for the core optimizations listed below, though they may utilize existing scene graphs or texture buffers (e.g., for baked shadows).

### Steps and Formulas

1.  **Early-Out Bounding Volumes**
    Surround expensive sub-scenes or complex geometry with a fast, simple SDF (typically a sphere). Before evaluating the complex SDF, check the bounding SDF. If the distance returned by the bounding SDF is greater than the current ray marching step size, skip the detailed evaluation entirely.
    *   Let $d_{bound}$ be the distance from the current ray point $P$ to the bounding volume.
    *   Let $d_{step}$ be the current step size (usually the SDF value of the active geometry).
    *   Condition: If $d_{bound} > d_{step}$, skip the complex SDF evaluation [S1, unofficial][S2, unofficial].

2.  **Level of Detail (LOD) Adjustment**
    Adjust the marching parameters based on the distance to the camera or the object. For distant geometry, reduce the maximum number of steps and increase the epsilon threshold. The visual difference is imperceptible beyond a certain distance (e.g., 10 scene units).
    *   Let $dist$ be the distance from the camera to the current ray point.
    *   If $dist > 10$, set $MAX\_STEPS_{distant} < MAX\_STEPS_{near}$ and $EPSILON_{distant} > EPSILON_{near}$ [S1, unofficial][S2, unofficial].

3.  **Resolution Scaling**
    Render the scene at a lower resolution (e.g., half resolution) and upscale the result. This is cited as the most effective single optimization for real-time SDF rendering at larger resolutions.
    *   Let $W, H$ be the target resolution.
    *   Render to buffer of size $\lfloor W/2 \rfloor \times \lfloor H/2 \rfloor$.
    *   Upscale using bilinear or nearest-neighbor filtering [S1, unofficial][S2, unofficial].

4.  **Over-Relaxation**
    Multiply the step size $d$ by a constant factor $k$ where $1.1 \le k \le 1.6$. This allows the ray to advance faster through empty space or open scenes.
    *   Update rule: $t_{new} = t_{old} + d \times k$.
    *   **Pitfall:** This risks artifacts near concave edges or thin geometry, as the ray may overshoot the surface if the step becomes too large relative to the local curvature [S1, unofficial].

5.  **Pre-marched Shadow and Ambient Occlusion Baking**
    For static volumes (e.g., clouds), pre-calculate shadow and ambient occlusion data into a 3D texture.
    *   **Baking Shadow:** Pre-march a shadow ray through the volume. Due to low volume resolution (e.g., 4 shadow samples per cloud sample), this can be done in real-time or pre-computed. This allows dynamic time-of-day changes [S3, unofficial].
    *   **Baking Ambient Occlusion:** Pre-march ambient occlusion into the volume texture to add shading without runtime cost [S3, unofficial].
    *   **Runtime Lookup:** Use the baked texture to retrieve shadow/occlusion values with a single `tex3D` lookup, ensuring perfect alignment with cloud geometry [S3, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_sdf_raymarch(ray_origin, ray_direction, scene_sdf, bounds_sdf, 
                          max_steps_near, max_steps_far, epsilon_near, epsilon_far,
                          use_over_relaxation, over_relaxation_factor, 
                          baked_shadow_tex, baked_ao_tex):
    """
    Performs ray marching with performance optimizations.
    """
    t = 0.0
    steps = 0
    
    # Determine initial LOD based on distance (simplified: assume near start)
    # In practice, this check happens inside the loop or per-object
    is_distant = False 
    
    while steps < max_steps_near:
        # Get current point
        p = ray_origin + ray_direction * t
        
        # 1. Early-Out Bounding Volumes
        # Check if we are inside a bounding volume that has a complex child
        # Simplified: Assume bounds_sdf returns distance to nearest complex object
        d_bound = bounds_sdf(p)
        
        # If the bounding volume is further than our current step would allow,
        # we might skip, but typically this is used to skip the complex SDF call
        # if the point is far from the complex object.
        # Here, we assume bounds_sdf is a cheap sphere check.
        # If d_bound is very large, we might skip detailed evaluation if we knew
        # we were outside the bounding box. For this generic implementation,
        # we proceed to the main SDF.
        
        # 2. Evaluate Main SDF
        d = scene_sdf(p)
        
        # 3. Level of Detail
        dist_to_camera = length(p - ray_origin)
        if dist_to_camera > 10.0:
            is_distant = True
            current_max_steps = max_steps_far
            current_epsilon = epsilon_far
        else:
            is_distant = False
            current_max_steps = max_steps_near
            current_epsilon = epsilon_near
            
        if steps > current_max_steps:
            break
            
        # 4. Over-Relaxation
        if use_over_relaxation:
            step_size = d * over_relaxation_factor
        else:
            step_size = d
            
        # Safety clamp to prevent infinite loops or negative steps
        if step_size < 0.0001:
            step_size = 0.0001
            
        t += step_size
        steps += 1
        
        # Check for hit
        if d < current_epsilon:
            # 5. Pre-marched Shadow/AO Lookup (if applicable)
            shadow = 1.0
            ao = 1.0
            if baked_shadow_tex is not None:
                shadow = tex3D_lookup(baked_shadow_tex, p)
            if baked_ao_tex is not None:
                ao = tex3D_lookup(baked_ao_tex, p)
                
            return hit(p, shadow, ao)
            
        # Check for miss (max distance)
        if t > 100.0: # Arbitrary max distance
            break
            
    return miss()
```

## Subtopics

### [Subsurface Scattering Approximation via Local Thickness](1946-subsurface-scattering-approximation-via-local-thickness.md)
Approximates subsurface scattering by computing the local thickness of a surface along random inward hemispherical directions, then using the thickness to modulate a simplified SSS lighting term. It is chosen when full SSS simulation is too expensive but some translucency is desired. See [Subsurface Scattering Approximation via Local Thickness](1946-subsurface-scattering-approximation-via-local-thickness.md).

### [Aggressive Ray Marching with SDF](2326-aggressive-ray-marching-with-sdf.md)
Uses SDF values to dynamically determine step sizes, allowing large skips in empty space and fine-grained sampling near surfaces. It is distinguished by its use of a configurable `ray_marching_ratio` to scale the SDF-derived step size and its integration with neural implicit representations and explicit termination masks. See [Aggressive Ray Marching with SDF](2326-aggressive-ray-marching-with-sdf.md).

### [Coverage Search](2534-coverage-search.md)
A SDF raymarching algorithm designed to solve ray hit/miss in fewer iterations than sphere tracing by using a stack to manage coverage spans along the ray. It is chosen when exact hit/miss determination is prioritized over iterative approximation. See [Coverage Search](2534-coverage-search.md).

### [SDF-Accelerated Volumetric Ray Marching](3681-sdf-accelerated-volumetric-ray-marching.md)
Accelerates volumetric ray marching by using an SDF to skip large empty spaces. The SDF stores the distance to the nearest surface of the volume, allowing the ray to advance by that distance in empty space. See [SDF-Accelerated Volumetric Ray Marching](3681-sdf-accelerated-volumetric-ray-marching.md).

### [Adaptive SDF Ray Marching with Cell Acceleration](3714-adaptive-sdf-ray-marching-with-cell-acceleration.md)
Marches a ray through an SDF scene using an acceleration structure to skip empty space, adaptive step sizing, and sign-change detection for surface refinement. See [Adaptive SDF Ray Marching with Cell Acceleration](3714-adaptive-sdf-ray-marching-with-cell-acceleration.md).

### [SDF Ray Marching with Eikonal Correction](4598-sdf-ray-marching-with-eikonal-correction.md)
Marches a ray through an SDF field while applying a correction factor to counteract distance elongation along non-cardinal directions caused by the Eikonal equation. See [SDF Ray Marching with Eikonal Correction](4598-sdf-ray-marching-with-eikonal-correction.md).

### [Ray Marching with SDFs and Soft Accumulation](5001-ray-marching-with-sdfs-and-soft-accumulation.md)
Renders hair-thin strands (e.g., jellyfish tentacles) using SDFs combined with soft accumulation to anti-alias across frames. See [Ray Marching with SDFs and Soft Accumulation](5001-ray-marching-with-sdfs-and-soft-accumulation.md).

### Pre-marched Shadow and Ambient Occlusion Baking
A technique to improve visual fidelity and performance by pre-calculating shadow and ambient occlusion data into the volume texture, leveraging the static nature of the volume. It involves pre-marching shadow and AO rays into a 3D texture, allowing runtime lookup with a single `tex3D` call. See [Pre-marched Shadow and Ambient Occlusion Baking](1946-subsurface-scattering-approximation-via-local-thickness.md) (Note: The source describes this under the context of SDF volumetric rendering, but the specific article link provided in the prompt for this variant is missing a dedicated URL. Based on the prompt's structure, this is a minor variant without its own article, so it is described here. However, the prompt lists specific children. I will treat this as a minor variant described in the notes but not having a dedicated child article link in the "Children" list, or it might be a sub-topic of one of the existing ones. The prompt says "Minor variants (they have no article of their own: write each one under Subtopics as a '### <Name>' entry carrying ALL of its details below, cited". So I will write it here.)

*Wait, the prompt lists specific children. "Pre-marched Shadow..." is listed under "Minor variants". It does not have a link in the "Children" list. I must write it under Subtopics.*

### Pre-marched Shadow and Ambient Occlusion Baking
A technique to improve visual fidelity and performance by pre-calculating shadow and ambient occlusion data into the volume texture, leveraging the static nature of the cloud volume. It involves pre-marching a shadow ray through the volume (e.g., 4 shadow samples per cloud sample) and pre-marching ambient occlusion into the volume texture. This allows dynamic time-of-day changes and runtime shadow lookup with a single `tex3D` lookup, matching the cloud geometry perfectly [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Reduced by a factor proportional to the number of skipped steps (via early-outs and over-relaxation) and the reduction in pixel count (via resolution scaling).
*   **Memory Complexity:** Minimal increase for LOD parameters. Baked shadow/AO textures increase memory usage proportional to the volume resolution [S3, unofficial].
*   **Measured Performance:** Rendering at half resolution is cited as the "most effective single optimisation for real-time SDF rendering at larger resolutions" [S1, unofficial][S2, unofficial].
*   **Strengths:** Simple to implement; significant FPS gains in real-time applications; effective for static scenes (baking).
*   **Limitations:** Over-relaxation can cause artifacts near concave edges [S1, unofficial]. Baking requires static geometry [S3, unofficial]. LOD adjustments may reduce visual quality for distant objects if not tuned correctly [S1, unofficial][S2, unofficial].
*   **When to use:** Real-time rendering where frame rate is critical; static volumetric scenes; scenes with large empty spaces.
*   **When not to use:** Dynamic scenes requiring high-fidelity shadows without pre-computation; scenes with very thin geometry where over-relaxation causes overshooting.

## Background
The techniques described are derived from practical implementations of ray marching in real-time graphics, particularly in the context of SDF rendering and volumetric clouds. The concepts of early-out bounding volumes, LOD, and resolution scaling are common in production graphics [S1, unofficial][S2, unofficial]. The specific application of pre-marching shadow and AO into volume textures for static clouds was detailed in a blog post by UHawkVR [S3, unofficial]. The term "sphere tracing" was introduced by John C. Hart in 1996 [S2, unofficial].

## Sources
[S1] Ray Marching and Signed Distance Functions — GLSL in the..., https://www.mysimulator.uk/content/articles/ray-marching-sdf.html, fetched 2026-10-29 (unofficial)
[S2] Ray Marching and Signed Distance Functions: Sphere Tracing..., https://www.mysimulator.uk/articles/ray-marching/, fetched 2026-10-29 (unofficial)
[S3] Rendering volumetric clouds using signed distance fields, https://blog.uhawkvr.com/rendering/rendering-volumetric-clouds-using-signed-distance-fields/, fetched 2026-10-26 (unofficial)
