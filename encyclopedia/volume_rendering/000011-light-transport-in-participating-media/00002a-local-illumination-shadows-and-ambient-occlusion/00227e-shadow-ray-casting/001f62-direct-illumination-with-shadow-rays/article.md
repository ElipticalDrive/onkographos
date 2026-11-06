# Direct Illumination with Shadow Rays

Also called: Direct Lighting with Shadows, Direct Lighting with Volume Transmittance, Direct Radiance with Volumetric Transmittance, Shadow Ray Illumination, Shadow Ray Lighting

## Overview
Computes the direct lighting contribution at a surface point or volume sample by casting shadow rays toward light sources, accounting for attenuation by heterogeneous participating media between the point and the light [S1, S2, unofficial]. It differs from pure surface shadowing by integrating volumetric transmittance along the ray path, and from full path tracing by only evaluating direct illumination from explicit light sources rather than indirect bounces [S1, S2, unofficial].

## How it works
**Inputs:**
- Hit point position $\mathbf{p}$ and surface normal $\mathbf{n}$ [S2, S3, unofficial].
- List of light sources, each with position $\mathbf{l}_i$, color $\mathbf{C}_i$, and enabled flag [S2, unofficial].
- Surface material properties (diffuse color, specular, roughness, metallic) [S2, S3, unofficial].
- Camera/view direction $\mathbf{V}$ for specular calculations [S3, S4, unofficial].

**Outputs:**
- Direct illumination radiance $\mathbf{L}_{direct}$ at $\mathbf{p}$ [S1, S2, unofficial].

**Data Structures:**
- Light iteration: loop over lights up to `MAX_LIGHTS` (typically 8) [S2, unofficial].
- Shadow ray result: intersection distance $t$ and hit status [S3, S4, unofficial].

**Steps:**
1. Initialize accumulated light color $\mathbf{L}_{direct} = \mathbf{0}$ [S2, unofficial].
2. For each light $i$ in the light list:
   a. If light $i$ is disabled, skip [S2, unofficial].
   b. Compute light direction $\mathbf{L} = \mathbf{l}_i - \mathbf{p}$ and distance $d = \|\mathbf{L}\|$ [S2, S4, unofficial].
   c. Normalize direction: $\mathbf{w}_i = \mathbf{L} / d$ [S1, unofficial].
   d. Create a shadow ray starting from $\mathbf{p} + \epsilon \mathbf{n}$ (where $\epsilon \approx 0.01$) in direction $\mathbf{w}_i$ [S3, unofficial].
   e. Compute volumetric transmittance $T_{light}$ from $\mathbf{p}$ to $\mathbf{l}_i$:
      - Integrate extinction over 16 uniform steps from $\mathbf{p}$ to $\mathbf{l}_i$ [S1, unofficial].
      - Alternatively, query a volume integrator interface: $T_{light} = \text{Transmittance}(\text{shadow\_ray})$ [S4, unofficial].
   f. Check geometric occlusion: if the shadow ray intersects geometry at distance $t < d$, set $T_{light} = 0.0$ [S4, unofficial]. Otherwise, if no intersection or $t \ge d$, visibility $V = 1$, else $V = 0$ [S1, unofficial].
   g. Compute geometric term:
      $$ G = \frac{\max(0, \mathbf{n} \cdot \mathbf{w}_i) \cdot |\mathbf{w}_i \cdot \mathbf{n}_{light}|}{d^2} $$
      where $\mathbf{n}_{light}$ is the light surface normal (for area lights) [S1, unofficial].
   h. Compute probability density $p_y = 1.0 / \text{light\_area}$ for importance sampling [S1, unofficial].
   i. Compute BRDF contribution:
      - Diffuse: $\mathbf{C}_i \cdot T_{light} \cdot V \cdot \frac{\rho_d}{\pi} \cdot G / p_y$ [S1, unofficial].
      - Specular (if enabled): $\mathbf{C}_i \cdot T_{light} \cdot \text{specular} \cdot \max(0, \mathbf{H} \cdot \mathbf{n})^{\text{shininess}}$, where $\mathbf{H} = \text{normalize}(\mathbf{V} + \mathbf{w}_i)$ and $\text{shininess} = 1/\text{roughness}$ [S3, S4, unofficial].
   j. Add contribution to $\mathbf{L}_{direct}$ [S2, unofficial].
3. Return $\mathbf{L}_{direct}$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dot(a, b): return sum(x*y for x, y in zip(a, b))
def length(v): return math.sqrt(dot(v, v))
def normalize(v): l = length(v); return [x/l for x in v] if l > 1e-8 else [0.0, 0.0, 0.0]
def sub(a, b): return [a[i]-b[i] for i in range(3)]
def add(a, b): return [a[i]+b[i] for i in range(3)]
def mul_scalar(v, s): return [x*s for x in v]
def mul_vec(a, b): return [a[i]*b[i] for i in range(3)]

class Ray:
    def __init__(self, origin, direction):
        self.origin = origin
        self.direction = direction

def direct_illumination_with_shadow_rays(hit_point, normal, view_dir, lights, volume_integrator, geometry_tracer):
    L_direct = [0.0, 0.0, 0.0]
    MAX_LIGHTS = 8  # Typical limit [S2, unofficial]
    
    for i in range(min(len(lights), MAX_LIGHTS)):
        light = lights[i]
        if not light.enabled:
            continue
            
        delta = sub(light.position, hit_point)
        d = length(delta)
        if d < 1e-6:
            continue
        w_i = normalize(delta)
        
        epsilon = 0.01 * length(normal)
        shadow_origin = add(hit_point, mul_scalar(normal, epsilon))
        shadow_ray = Ray(shadow_origin, w_i)
        
        T_light = volume_integrator.Transmittance(shadow_ray)
        
        intersection = geometry_tracer.intersect(shadow_ray)
        if intersection.hit and intersection.t < d:
            T_light = 0.0
        V = 1.0 if (not intersection.hit or intersection.t >= d) else 0.0
        
        cos_theta = max(0.0, dot(normal, w_i))
        cos_light = 1.0 if light.is_point else max(0.0, dot(mul_scalar(w_i, -1.0), light.normal))
        geometric_term = (cos_theta * cos_light) / (d * d)
        
        p_y = 1.0 / light.area if light.area > 0 else 1.0
        
        # Diffuse contribution
        L_direct = add(L_direct, mul_vec(light.color, mul_scalar([T_light, T_light, T_light], V * geometric_term / p_y)))
        L_direct = mul_vec(L_direct, hit_material.diffuse_color)
        
        # Specular contribution
        if hit_material.specular_enabled:
            H = normalize(add(view_dir, w_i))
            cos_Hn = max(0.0, dot(H, normal))
            shininess = 1.0 / hit_material.roughness
            specular_scale = T_light * hit_material.specular * (cos_Hn ** shininess)
            L_direct = add(L_direct, mul_vec(light.color, [specular_scale, specular_scale, specular_scale]))
            
    return L_direct
```

**Parameters:**
- $\epsilon$: Shadow ray offset, typically $0.01 \times \|\mathbf{n}\|$ [S3, unofficial].
- Number of integration steps: 16 for discrete integration [S1, unofficial].
- MAX_LIGHTS: Maximum number of lights, typically 8 [S2, unofficial].

**Edge Cases and Pitfalls:**
- Precision: Shadow ray origin must be offset to avoid self-intersection [S3, unofficial].
- Boundaries: Handle light distance $d=0$ to avoid division by zero [S2, unofficial].
- Degenerate input: Disabled lights should be skipped early [S2, unofficial].

## Subtopics

### Transparent Shadows
A shadowing technique where shadow rays cross smooth glass in a straight line, and paths refracted to a light already reached that way do not count as reaching the light [S5, unofficial].

### [Shadow Ray Transmittance Estimation](5248-shadow-ray-transmittance-estimation.md)
Estimates transmittance from a point to a distant light source using majorant sampling to handle participating media. Not given in the sources for algorithmic details beyond the voxel direct lighting example.

## Operating characteristics
- **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of lights and $S$ is the number of sampling steps per shadow ray [S1, unofficial].
- **Memory Complexity:** $O(1)$ per light, plus volume grid storage [S6, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Linear with number of lights and volume samples [S1, unofficial].
- **Strengths:** Handles heterogeneous volumes, supports multiple lights, integrates with PBR [S1, S2, unofficial].
- **Limitations:** Not given in the sources.
- **When to use:** When volumetric shadows are required and direct illumination is sufficient [S1, unofficial].
- **When not to use:** Not given in the sources.

## Background
Originated in volumetric path tracing implementations to handle direct lighting in participating media [S1, unofficial]. Used in real-time ray marchers and offline path tracers for volumetric effects [S2, S6, unofficial].

## Sources
[S1] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
[S2] unfinishedprogram/ray-marcher, https://github.com/unfinishedprogram/ray-marcher, fetched 2026-10-26 (unofficial)
[S3] An Introduction to Raymarching - Maxime Garcia, https://typhomnt.github.io/teaching/ray_tracing/raymarching_intro/, fetched 2026-10-27 (unofficial)
[S4] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
[S5] Milestone 13: MagicaVoxel look (.vox, DDA voxel grid, sun/sky, bloom, style voxel), https://github.com/adriendjn/mypbr/pull/10, fetched 2026-10-25 (unofficial)
[S6] ria8651/bevy-voxel-engine, https://github.com/ria8651/bevy-voxel-engine, fetched 2026-10-27 (unofficial)
