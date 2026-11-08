# Customized Cone Tracing

Also called: Voxel Cone Tracing with modifications

## Overview
Customized Cone Tracing is a variant of Voxel Cone Tracing that modifies the standard occlusion, specular, and diffuse cone behaviors to improve aesthetic quality and mitigate specific artifacts inherent to the standard model [S1, unofficial]. It addresses issues such as overly dark corners in occlusion, unrealistic specular highlights from self-intersection on curved surfaces, and dim indirect diffuse lighting [S1, unofficial]. It sets itself apart from standard Voxel Cone Tracing by replacing the generic accumulation and BRDF models with specialized, experimentally tuned functions for each cone type [S1, unofficial].

## How it works
Customized Cone Tracing modifies the standard cone tracing loop for three specific cone types: Occlusion, Specular, and Diffuse [S1, unofficial]. The method samples voxel data containing `a` (occlusion/alpha) and `color` fields [S1, unofficial].

### Inputs and Outputs
*   **Input:** Voxel data (fields `a` and `color`), surface point position, light direction, material parameters (Phong shininess $\alpha$, shininess_falloff $q$), and cone parameters (`max_dist`, `shadow_str`) [S1, unofficial].
*   **Output:** Accumulated occlusion factor, specular color contribution, and diffuse color contribution [S1, unofficial].

### Data Structures
The voxel structure stores `a` (float, occlusion value) and `color` (vec3, sampled color) [S1, unofficial].

### Parameters
*   `shadow_str`: Float, occlusion strength multiplier [S1, unofficial].
*   `max_dist`: Float, maximum tracing distance for the cone [S1, unofficial].
*   `p`: Float, angular grace value. Defined as $p = 0.008 \cdot \pi \cdot \alpha$, where $\alpha$ is the Phong shininess (range 0-256) [S1, unofficial].
*   `q`: Float, shininess_falloff exponent. Included in `VoxSettings` [S1, unofficial].
*   `bias`: Float, bias for diffuse cones [S2, unofficial].
*   `reflection_bias`: Float, bias for specular/reflection cones [S2, unofficial].

### Algorithm Steps

#### 1. Occlusion Cone Tracing
The occlusion cone computes shadows and ambient occlusion [S1, unofficial][S4, unofficial]. To mitigate the intensity of short-distance occlusion (which causes overly dark corners), sampled values are decreased proportional to the square root of the distance traveled, smoothed via Hermite interpolation (`smoothstep`) [S1, unofficial].

1.  Initialize `occ = 0.0`.
2.  March the cone from the surface point towards the light source (or hemisphere for AO) [S1, unofficial][S4, unofficial].
3.  At each step, sample the voxel at the current position. Let `voxel.a` be the sampled alpha [S1, unofficial].
4.  Calculate the current distance traveled, `current_dist` [S1, unofficial].
5.  Compute the intermediate occlusion value `occ_r`:
    $$ occ_r = voxel.a \cdot \text{smoothstep}(0.0, max\_dist, \sqrt{current\_dist} \cdot shadow\_str) $$
    Here, `smoothstep` acts as the Hermite interpolation function [S1, unofficial].
6.  Update the accumulated occlusion:
    $$ occ = occ + (1 - occ) \cdot occ_r $$
7.  Return `occ` as the final occlusion factor [S1, unofficial].

#### 2. Specular Cone Tracing
The specular cone is traced in the direction of the reflected view vector [S3, unofficial]. To prevent unrealistic highlights caused by the cone intersecting the surface it originated from (self-intersection on curved surfaces), a specialized BRDF is used that depends on the angle between the cone travel direction and the light source direction [S1, unofficial].

1.  Initialize `spec_color = (0, 0, 0)` [S1, unofficial].
2.  March the cone in the direction of the reflected view vector $\vec{\omega}$ [S3, unofficial].
3.  Let $\vec{l}$ be the direction to the light source [S1, unofficial].
4.  Calculate the angle $\gamma$ between $\vec{\omega}$ and $\vec{l}$:
    $$ \gamma = \arccos(\vec{\omega} \cdot \vec{l}) $$
5.  Calculate the normalized angle $\gamma_n$ using the angular grace value $p$:
    $$ \gamma_n = \max(0, \gamma - p) $$
6.  Compute the specular BRDF factor $f_{spec}$:
    $$ f_{spec} = \left(1 - \frac{\gamma_n}{\pi}\right)^q $$
    Where $q$ is the shininess_falloff [S1, unofficial].
7.  Accumulate the color:
    $$ spec\_color += voxel.color \cdot f_{spec} \cdot \text{weight} $$
    Weighting depends on the specific marching routine, typically related to cone volume or distance [G].
8.  Return `spec_color` [S1, unofficial].

#### 3. Diffuse Cone Tracing
The diffuse cone traces into the hemisphere to capture indirect lighting [S1, unofficial]. Tracing diffuse cones often resulted in very dim indirect diffuse light, so a crude stopping condition is applied: if the sampled volume occlusion exceeds 1/100, the trace stops immediately and returns the color sampled at that point [S1, unofficial].

1.  Initialize `diff_color = (0, 0, 0)` [S1, unofficial].
2.  March the cone into the hemisphere [S1, unofficial].
3.  At each step, sample the voxel. Let `voxel.a` be the sampled alpha [S1, unofficial].
4.  Check if `voxel.a > 1/100` (0.01) [S1, unofficial].
5.  If true, stop the cone trace immediately and return the `voxel.color` sampled at that point [S1, unofficial].
6.  If false, accumulate the color:
    $$ diff\_color += voxel.color \cdot \text{weight} $$
    Weighting depends on the specific marching routine [G].
7.  Return `diff_color` [S1, unofficial].

#### 4. Final Computation
The final lighting at a point is computed by combining direct and indirect components [S1, unofficial].

1.  Compute Phong-based direct lighting $L_{direct}$ using the surface normal, view vector, and light direction [S1, unofficial].
2.  Trace a shadow-cone towards the light source to get the occlusion factor $O$ using the Occlusion Cone algorithm [S1, unofficial].
3.  Compute final color:
    $$ C_{final} = L_{direct} \cdot O + L_{indirect} $$
    Where $L_{indirect}$ is the sum of contributions from the Diffuse and Specular cones [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def smoothstep(edge0, edge1, x):
    t = max(0.0, min(1.0, (x - edge0) / (edge1 - edge0)))
    return t * t * (3.0 - 2.0 * t)

def customized_cone_trace_occlusion(voxel_tree, start_pos, direction, max_dist, shadow_str):
    occ = 0.0
    current_dist = 0.0
    step = max_dist / 32.0  # Standard marching step size [G]
    while current_dist < max_dist:
        voxel = voxel_tree.sample(start_pos + direction * current_dist)
        t = math.sqrt(current_dist) * shadow_str
        h = smoothstep(0.0, max_dist, t)
        occ_r = voxel.a * h
        occ = occ + (1.0 - occ) * occ_r
        current_dist += step
    return occ

def customized_cone_trace_specular(voxel_tree, start_pos, reflected_view_dir, light_dir, alpha, q):
    spec_color = [0.0, 0.0, 0.0]
    current_dist = 0.0
    step = max_dist / 32.0  # Standard marching step size [G]
    p = 0.008 * math.pi * alpha
    while current_dist < max_dist:
        voxel = voxel_tree.sample(start_pos + reflected_view_dir * current_dist)
        dot_val = max(-1.0, min(1.0, dot(reflected_view_dir, light_dir)))
        gamma = math.acos(dot_val)
        gamma_n = max(0.0, gamma - p)
        f_spec = pow(1.0 - (gamma_n / math.pi), q)
        weight = 1.0 / (1.0 + current_dist)  # Standard cone volume weighting [G]
        spec_color[0] += voxel.color[0] * f_spec * weight
        spec_color[1] += voxel.color[1] * f_spec * weight
        spec_color[2] += voxel.color[2] * f_spec * weight
        current_dist += step
    return spec_color

def customized_cone_trace_diffuse(voxel_tree, start_pos, hemisphere_samples):
    diff_color = [0.0, 0.0, 0.0]
    for sample in hemisphere_samples:
        current_dist = 0.0
        step = max_dist / 32.0  # Standard marching step size [G]
        while current_dist < max_dist:
            voxel = voxel_tree.sample(start_pos + sample.direction * current_dist)
            if voxel.a > 0.01:
                return voxel.color
            weight = 1.0 / (1.0 + current_dist)  # Standard cone volume weighting [G]
            diff_color[0] += voxel.color[0] * weight
            diff_color[1] += voxel.color[1] * weight
            diff_color[2] += voxel.color[2] * weight
            current_dist += step
    return diff_color
```

## Subtopics

### [Two-Bounce Indirect Illumination](4558-two-bounce-indirect-illumination.md)
A three-step GPU-based algorithm for computing two light bounces of indirect illumination for Lambertian and glossy materials at interactive framerates. It differs from standard Ambient Occlusion by capturing and storing incoming radiance direction and energy in a sparse voxel octree, enabling view-dependent specular reflections and color bleeding rather than just visibility. It builds on Customized Cone Tracing by explicitly separating the capture of direct illumination from the final gathering phase.

### [Multi-Cone Shading](5176-multi-cone-shading.md)
Multi-Cone Shading is a real-time approximation technique for diffuse interreflection in participating media, specifically designed to improve the quality of indirect lighting computed via Voxel Cone Tracing. While standard cone tracing samples a single direction (typically the surface normal), Multi-Cone Shading casts five cones in distinct directions from a surface point and averages the results to approximate diffuse lighting from multiple angles. It distinguishes itself from its sibling [Two-Bounce Indirect Illumination](4558-two-bounce-indirect-illumination.md) by focusing on the angular distribution of the initial diffuse bounce rather than simulating secondary bounces, and from [Diffuse Cone Distribution](5677-diffuse-cone-distribution.md) by using a fixed, discrete set of five directions rather than a continuous distribution model.

### [Diffuse Cone Distribution](5677-diffuse-cone-distribution.md)
Diffuse Cone Distribution is a sampling strategy for approximating indirect diffuse lighting in Voxel Cone Tracing. It uses a fixed, deterministic arrangement of five cones—one aligned with the surface normal and four at perpendicular angles—distinguishing it from stochastic rotation or multi-bounce variants.

### Separate Diffuse/Reflection Bias
A parameter separation technique where the single cone tracing bias value is split into two independent parameters: `bias` for diffuse cones and `reflection_bias` for the single specular/reflection cone [S2, unofficial]. This allows independent tuning because reflections benefit from a smaller bias for sharp contact reflections, while diffuse GI benefits from a larger bias to reduce self-occlusion noise [S2, unofficial]. It is implemented in the RD (Forward+/Mobile) backend, which is the only backend that actually implements VoxelGI cone tracing [S2, unofficial]. The `VoxelGIData` uniform struct byte layout is duplicated in 4 places (`gi.h`, `gi.glsl`, `scene_forward_clustered_inc.glsl`, `volumetric_fog_process.glsl`) and must be updated in lockstep to keep the `std140` layout identical [S2, unofficial].

### Specular Cone Tracing
A variant where a cone is cast in the direction of the reflected view vector [S3, unofficial]. The cone aperture depends on surface roughness: high roughness results in a high aperture, while smooth surfaces result in a small aperture [S3, unofficial]. Smooth surfaces are more expensive to calculate because smaller apertures require more marching steps [S3, unofficial].

### Ambient Occlusion via Cone Tracing
Ambient occlusion is approximated using the same cones used for diffuse reflection for efficiency [S4, unofficial]. Only the occlusion value $\alpha_2$ is accumulated [S4, unofficial]. A weighting function $f(r) = \frac{1}{1+\lambda r}$ is applied at each step, where $r$ is the current radius of the cone and $\lambda$ is a user-defined value controlling the decay rate [S4, unofficial]. The accumulated value is updated as $\delta = \delta + (1-\delta)\alpha_2f(r)$ [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of cone steps [S1, unofficial]. The customized specular BRDF involves trigonometric functions (`arccos`, `pow`) per step, which is more expensive than simple linear interpolation but cheaper than full ray tracing [S1, unofficial].
*   **Memory Complexity:** Depends on the sparse voxel octree size [S2, unofficial]. The Separate Diffuse/Reflection Bias variant increases memory usage slightly due to the need for anisotropic voxel mipmaps (6 directional copies) in the RD backend, costing roughly one extra full copy of the base mip chain's memory [S2, unofficial].
*   **Strengths:** Mitigates specific visual artifacts (dark corners, unrealistic specular highlights) better than standard Voxel Cone Tracing [S1, unofficial]. The early termination in diffuse cones can save computation time in occluded areas [S1, unofficial].
*   **Limitations:** The customized specular BRDF is view-independent in its formulation (depending only on cone direction and light direction), which may not perfectly match view-dependent specular highlights in all cases [S1, unofficial]. The "crude" diffuse stopping condition may lead to discontinuities if not smoothed [S1, unofficial]. Smooth surfaces are more expensive to calculate due to smaller cone apertures requiring more marching steps [S3, unofficial].
*   **When to use:** When aesthetic quality of shadows and specular highlights is prioritized over physical accuracy, and when standard Voxel Cone Tracing produces artifacts like overly dark corners or grey highlights on curved surfaces [S1, unofficial].

## Background
Customized Cone Tracing was developed as part of a thesis on real-time global illumination using OpenGL and Voxel Cone Tracing [S1, unofficial]. The method emerged from a "long process of experimentation" to achieve aesthetically pleasing results, modifying the standard algorithm established in prior work (referenced as 4.3.2 in the source) [S1, unofficial]. It is primarily associated with the Godot Engine's VoxelGI implementation, where further modifications like anisotropic voxel mipmaps and separate bias parameters were added [S2, unofficial].

## Sources
[S1] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26 (unofficial)
[S2] Add anisotropic voxel mipmaps and separate diffuse/reflection bias to VoxelGI, https://github.com/NeoCry/godot_landscape/pull/12, fetched 2026-10-26 (unofficial)
[S3] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
[S4] Deferred Voxel Shading for Real Time Global Illumination, https://jose-villegas.github.io/post/deferred_voxel_shading/, fetched 2026-10-07 (unofficial)
