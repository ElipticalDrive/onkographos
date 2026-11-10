# Direct Illumination with Shadow Rays

Also called: Direct Lighting with Shadows, Direct Lighting with Volume Transmittance, Direct Radiance with Volumetric Transmittance, Shadow Ray Illumination, Shadow Ray Lighting

## Overview
Direct Illumination with Shadow Rays computes the direct lighting contribution at a surface point by evaluating visibility and volumetric attenuation toward light sources [S1, S2, unofficial]. It differs from [Ambient Occlusion via Shadow Rays](878-ambient-occlusion-via-shadow-rays.md) by calculating specific directional light contributions rather than a hemisphere-integrated ambient term, and from [Lookup Table Shadow Rendering](3588-lookup-table-shadow-rendering.md) by performing real-time ray traversal instead of precomputed texture lookups.

## How it works
**Inputs:** Surface position $\mathbf{p}$, surface normal $\mathbf{n}$, list of light sources $\{L_i\}$, volume density field $\sigma_t(\mathbf{x})$ (or geometry for binary shadowing), and material properties (diffuse/specular coefficients).
**Outputs:** Direct radiance $L_{direct}$ at $\mathbf{p}$.

**Data Structures:**
*   **Light Source:** Contains position $\mathbf{p}_L$, color $C_L$, and optionally area $A_L$ [S1, S2, unofficial].
*   **Shadow Ray Result:** Boolean or transmittance value $T \in [0, 1]$ indicating occlusion [S1, S2, unofficial].

**Steps:**
1.  **Initialization:** Initialize accumulated direct light color $L_{direct} = \mathbf{0}$ [S2, unofficial].
2.  **Light Loop:** For each light source $L_i$:
    a.  **Direction:** Compute light direction vector $\mathbf{l} = \mathbf{p}_L - \mathbf{p}$ and distance $d = \|\mathbf{l}\|$ [S2, unofficial]. Normalize to get $\hat{\mathbf{l}} = \mathbf{l}/d$ [S4, unofficial].
    b.  **Shadow Ray Casting:** Cast a shadow ray from $\mathbf{p}$ in direction $\hat{\mathbf{l}}$ [S4, unofficial].
        *   *Offset:* To avoid self-intersection, start the ray at $\mathbf{p}' = \mathbf{p} + \epsilon \mathbf{n}$, where $\epsilon$ is a small constant (e.g., $0.01$) [S3, G, unofficial].
        *   *Traversal:* March along the ray to detect occlusion [S3, unofficial].
        *   *Visibility Check:* Determine if the ray hits an occluder before reaching the light [S3, unofficial].
            *   If no hit occurs, or the hit distance $t_{hit} \ge d$, the light is visible ($V=1$) [S3, unofficial].
            *   If an occluder is found at $t_{hit} < d$, the light is shadowed ($V=0$) [S3, S4, unofficial].
        *   *Volumetric Transmittance:* If the scene contains participating media, compute transmittance $T_{vol}$ along the path from $\mathbf{p}$ to $\mathbf{p}_L$ [S1, unofficial]. This is typically done by integrating the extinction coefficient $\sigma_t$ along the ray [S1, unofficial].
            $$ T_{vol} = \exp\left(-\int_{0}^{d} \sigma_t(\mathbf{p} + s\hat{\mathbf{l}}) \, ds\right) $$
            In discrete implementations, this integral is approximated by summing $\sigma_t \cdot \Delta s$ over $N$ samples (e.g., $N=16$) [S1, unofficial].
    c.  **Lighting Contribution:**
        *   Compute the geometric term $G$ [S1, unofficial]:
            $$ G = \frac{\max(0, \mathbf{n} \cdot \hat{\mathbf{l}}) \cdot |\hat{\mathbf{l}} \cdot \mathbf{n}_L|}{d^2} $$
            where $\mathbf{n}_L$ is the normal of the light source (if applicable) [S1, unofficial]. For point lights, the second term is often 1.
        *   Compute the probability density $p_y$ for the light sampling [S1, unofficial]:
            $$ p_y = \frac{1}{A_L} $$
            where $A_L$ is the area of the light source [S1, unofficial].
        *   Calculate the BRDF contribution $L_{BRDF}$ using the light color $C_L$, transmittance $T_{vol}$, and visibility $V$ [S1, S4, unofficial]:
            $$ L_{BRDF} = \frac{C_L \cdot f_r(\mathbf{n}, \hat{\mathbf{l}}, -\hat{\mathbf{v}}) \cdot V \cdot T_{vol} \cdot G}{p_y} $$
            where $f_r$ is the BRDF (diffuse + specular) [S4, unofficial].
        *   Add to accumulator: $L_{direct} += L_{BRDF}$ [S2, unofficial].
3.  **Return:** Return $L_{direct}$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def direct_illumination_with_shadow_rays(p, n, lights, volume_integrator):
    """
    Computes direct lighting for a surface point p with normal n.
    lights: list of Light objects with position, color, and area.
    volume_integrator: object with method transmittance(ray) returning T.
    """
    L_direct = vec3(0.0)
    
    for light in lights:
        # 1. Compute light direction and distance
        delta = light.position - p
        d = length(delta)
        if d == 0:
            continue
        l_dir = normalize(delta)
        
        # 2. Cast shadow ray
        # Offset start point to avoid self-intersection [S3, unofficial]
        ray_origin = p + 0.01 * n
        shadow_ray = Ray(ray_origin, l_dir)
        
        # Check for occlusion
        # Implementation depends on scene representation (geometry vs volume)
        # Here we assume a generic intersection test or volume march
        hit_dist = scene_intersect(shadow_ray) # Returns distance to nearest hit, or infinity
        
        is_visible = False
        if hit_dist == infinity or hit_dist >= d:
            is_visible = True
        
        # 3. Compute Volumetric Transmittance [S1, unofficial]
        # Integrate sigma_t along the path to the light
        # Using 16-step integration as per [S1, unofficial]
        T_vol = volume_integrator.transmittance(shadow_ray, d)
        
        # 4. Compute Lighting Contribution [S1, S4, unofficial]
        if is_visible:
            # Geometric term [S1, unofficial]
            # Assuming point light, |dot(-wi, light_n)| is 1
            cos_theta = max(0.0, dot(n, l_dir))
            G = cos_theta / (d * d)
            
            # Probability density for area light [S1, unofficial]
            p_y = 1.0 / light.area
            
            # BRDF calculation [S4, unofficial]
            # Simplified diffuse BRDF: color / pi
            # Specular term omitted for brevity, but included in full PBR [S4, unofficial]
            brdf = light.color * (1.0 / 3.14159) # Diffuse only example
            
            # Final contribution
            L_direct += brdf * G * T_vol / p_y
        else:
            # Shadowed: add ambient occlusion term as fallback [S3, unofficial]
            L_direct += vec3(0.03) * hit.material.color * hit.material.ao
            
    return L_direct
```

**Parameters:**
*   $\epsilon$: Shadow ray offset scale. Typical value: $0.01$ [S3, unofficial].
*   $N$: Number of samples for volumetric integration. Typical value: $16$ [S1, unofficial].
*   $MAX\_LIGHTS$: Maximum number of lights processed. Typical value: $8$ [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Self-Intersection:** Failing to offset the shadow ray origin causes false occlusion. Use $\epsilon \mathbf{n}$ [S3, unofficial].
*   **Precision:** Floating-point errors in distance comparisons ($t_{hit} \ge d$) can cause flickering. Use a small epsilon tolerance [S3, unofficial].
*   **Volumetric Sampling:** Insufficient samples in the volume integration lead to noise. Adaptive sampling or majorant sampling may be required for heterogeneous media [S1, unofficial].

## Subtopics

### [Shadow Ray Transmittance Estimation](5248-shadow-ray-transmittance-estimation.md)
Estimates transmittance from a surface point to a distant light through participating media using majorant sampling. It differs from standard majorant sampling by integrating the sampling loop directly into the shadow ray traversal, allowing for tighter bounds per channel in chromatic variants and reducing variance caused by global majorants.

### [Voxel Direct Lighting and Shadowing](6610-voxel-direct-lighting-and-shadowing.md)
Computes local illumination and shadows in voxel-based scenes by casting shadow rays from surface positions toward light sources [G]. It differs from [Shadow Ray Transmittance Estimation](5248-shadow-ray-transmittance-estimation.md) by using binary hit/miss results for shadowing rather than estimating transmittance through participating media [G]. It builds on [Direct Illumination with Shadow Rays](2011-direct-illumination-with-shadow-rays.md) by applying these concepts specifically to voxel geometry [G].

### Transparent Shadows
A shadowing technique where shadow rays cross smooth glass in a straight line, and paths refracted to a light already reached that way do not count as reaching the light [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N_L \cdot N_s)$, where $N_L$ is the number of lights and $N_s$ is the number of samples per shadow ray [S1, S2, unofficial].
*   **Memory Complexity:** $O(1)$ per light, assuming the volume/geometry is accessed on-demand [S1, S4, unofficial].
*   **Strengths:** Accurate shadows and volumetric attenuation; simple to implement for point and directional lights [S1, S2, unofficial].
*   **Limitations:** High computational cost for many lights or complex volumes; noisy results in highly heterogeneous media without advanced sampling [S1, unofficial].
*   **When to Use:** Real-time rendering with limited lights, or offline rendering where accuracy is prioritized over speed [S2, S4, unofficial].
*   **When Not to Use:** Scenes with thousands of lights or extremely large volumes where precomputed shadows are more efficient [S4, unofficial].

## Background
Direct illumination with shadow rays is a fundamental technique in ray tracing and ray marching, originating from the work of Whitted (1980) on ray tracing [G]. It has been widely adopted in real-time graphics for local illumination [S2, S3, unofficial] and in offline rendering for accurate volumetric effects [S1, S4, unofficial].

## Sources
[S1] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S2] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S3] An Introduction to Raymarching - Maxime Garcia, https://typhomnt.github.io/teaching/ray_tracing/raymarching_intro/, fetched 2026-10-27 (unofficial)
[S4] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
[S5] Milestone 13: MagicaVoxel look (.vox, DDA voxel grid, sun/sky, bloom, style voxel), https://github.com/adriendjn/mypbr/pull/10, fetched 2026-10-25 (unofficial)
