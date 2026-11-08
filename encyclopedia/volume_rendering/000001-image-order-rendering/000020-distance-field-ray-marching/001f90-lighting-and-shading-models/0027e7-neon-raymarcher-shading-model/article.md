# Neon Raymarcher Shading Model

Also called: Milestone 2 shading, Procedural Neon Shading

## Overview
The Neon Raymarcher Shading Model is a real-time shading technique for procedural neon city scenes that computes diffuse, specular, ambient occlusion, metallic reflection, and emissive contributions per pixel [S1, unofficial]. It distinguishes itself by integrating a metalness-driven specular model and a Fresnel-weighted metallic reflection term tailored for high-contrast, emissive aesthetics [S1, unofficial].

## How it works
**Inputs:**
*   Surface point $\mathbf{p}$ in world space [S1, unofficial].
*   Surface normal $\mathbf{n}$, normalized [S1, unofficial].
*   Ray direction $\mathbf{rd}$ (camera to surface), normalized [S1, unofficial].
*   Light direction $\mathbf{l}$ (surface to light), normalized [S1, unofficial].
*   Light color $\mathbf{c}_{light}$ [S1, unofficial].
*   Sky color function $\mathbf{c}_{sky}(\cdot)$ [S1, unofficial].
*   Material properties: albedo $\mathbf{c}_{albedo}$, emissive $\mathbf{c}_{emissive}$, metalness $m \in [0, 1]$, and ambient light color/intensity $\mathbf{c}_{ambient}$ [S1, unofficial].

**Outputs:**
*   Final pixel color $\mathbf{c}_{out}$ [S1, unofficial].

**Data Structures:**
No complex data structures are required beyond standard vector and scalar types; the method relies on helper functions `calcAO` and `Shadow` [S1, unofficial].

**Steps:**

1.  **Geometric Setup:**
    Calculate the half-vector $\mathbf{h}$ for specular calculation:
    $$ \mathbf{h} = \frac{\mathbf{l} - \mathbf{rd}}{\|\mathbf{l} - \mathbf{rd}\|} $$
    [S1, unofficial]

2.  **Shadow Calculation:**
    Compute the shadow factor $s$ by casting a shadow ray from an offset position along the normal:
    $$ \mathbf{p}_{shadow} = \mathbf{p} + \mathbf{n} \cdot 0.03 $$
    $$ s = \text{Shadow}(\mathbf{p}_{shadow}, \mathbf{l}, 12.0) $$
    The offset of $0.03$ units prevents self-shadowing artifacts [S1, unofficial]. The maximum ray distance is $12.0$ units [S1, unofficial].

3.  **Ambient Occlusion:**
    Compute the ambient occlusion factor $a_o$ using `calcAO(p, n)` [S1, unofficial].

4.  **Diffuse Component:**
    Calculate the diffuse lighting contribution $\mathbf{c}_{diff}$:
    $$ d = \max(\mathbf{n} \cdot \mathbf{l}, 0.0) \cdot s $$
    $$ \mathbf{c}_{diff} = \mathbf{c}_{albedo} \cdot \mathbf{c}_{light} \cdot d $$
    [S1, unofficial]

5.  **Specular Component:**
    Calculate the specular intensity using a Blinn-Phong model with metalness-dependent exponent and intensity:
    $$ \text{spec} = \max(\mathbf{n} \cdot \mathbf{h}, 0.0)^{\text{mix}(32.0, 140.0, m)} \cdot s $$
    $$ \mathbf{c}_{spec} = \mathbf{c}_{light} \cdot \text{spec} \cdot \text{mix}(0.4, 2.0, m) $$
    [S1, unofficial]

6.  **Metallic Reflection:**
    Calculate the Fresnel-like term for metallic reflection:
    $$ f = (1.0 - \max(\mathbf{n} \cdot (-\mathbf{rd}), 0.0))^5 $$
    [S1, unofficial]
    
    Compute the reflected ray direction:
    $$ \mathbf{r} = \text{reflect}(\mathbf{rd}, \mathbf{n}) $$
    [S1, unofficial]
    
    Sample the sky color for the reflection:
    $$ \mathbf{c}_{refl} = \mathbf{c}_{sky}(\mathbf{r}) $$
    [S1, unofficial]
    
    Calculate the metallic reflection contribution $\mathbf{c}_{metal}$:
    $$ \mathbf{c}_{metal} = \mathbf{c}_{refl} \cdot \text{mix}(0.03, 0.6, m) \cdot (0.25 + 0.75 \cdot f) \cdot a_o $$
    [S1, unofficial]

7.  **Ambient Component:**
    Calculate the ambient lighting contribution $\mathbf{c}_{amb}$:
    $$ \mathbf{c}_{amb} = \mathbf{c}_{albedo} \cdot \mathbf{c}_{ambient} \cdot a_o $$
    [S1, unofficial]

8.  **Color Accumulation:**
    Combine all components:
    $$ \mathbf{c}_{out} = \mathbf{c}_{amb} + \mathbf{c}_{diff} + \mathbf{c}_{spec} + \mathbf{c}_{metal} + \mathbf{c}_{emissive} $$
    [S1, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mix(a, b, t):
    return a + t * (b - a)

def neon_raymarcher_shading(p, n, rd, l, c_light, c_sky, c_albedo, c_emissive, metal, c_ambient):
    # 1. Geometric Setup
    half_vec = (l - rd) / norm(l - rd)
    
    # 2. Shadow Calculation
    p_shadow = p + n * 0.03
    shadow_factor = shadow_ray(p_shadow, l, max_distance=12.0)
    
    # 3. Ambient Occlusion
    ao_factor = calc_ao(p, n)
    
    # 4. Diffuse Component
    diff = max(dot(n, l), 0.0) * shadow_factor
    c_diff = c_albedo * c_light * diff
    
    # 5. Specular Component
    spec_pow = mix(32.0, 140.0, metal)
    spec_int = mix(0.4, 2.0, metal)
    spec = pow(max(dot(n, half_vec), 0.0), spec_pow) * shadow_factor
    c_spec = c_light * spec * spec_int
    
    # 6. Metallic Reflection
    fresnel_val = 1.0 - max(dot(n, -rd), 0.0)
    fresnel_term = pow(fresnel_val, 5.0)
    
    r = reflect(rd, n)
    c_refl = c_sky(r)
    
    metal_int = mix(0.03, 0.6, metal)
    grazing_boost = 0.25 + 0.75 * fresnel_term
    c_metal = c_refl * metal_int * grazing_boost * ao_factor
    
    # 7. Ambient Component
    c_amb = c_albedo * c_ambient * ao_factor
    
    # 8. Color Accumulation
    c_out = c_amb + c_diff + c_spec + c_metal + c_emissive
    
    return c_out

# Helper: Shadow ray marching [G]
def shadow_ray(origin, direction, max_distance):
    t = 0.0
    while t < max_distance:
        t += step_size # step_size depends on scene scale
        if signed_distance_function(origin + direction * t) < 0.0:
            return 0.0
    return 1.0

# Helper: Ambient occlusion estimation [G]
def calc_ao(point, normal):
    # Standard SDF-based AO: sample along normal, accumulate occlusion
    ao = 1.0
    for i in range(1, 5):
        sample_point = point + normal * (i * ao_scale)
        if signed_distance_function(sample_point) < 0.0:
            ao -= ao_weight
    return max(ao, 0.0)
```

**Parameters:**
*   Shadow offset: $0.03$ [S1, unofficial].
*   Shadow max distance: $12.0$ [S1, unofficial].
*   Specular exponent range: $32.0$ to $140.0$ [S1, unofficial].
*   Specular intensity range: $0.4$ to $2.0$ [S1, unofficial].
*   Fresnel power: $5.0$ [S1, unofficial].
*   Metallic reflection base intensity range: $0.03$ to $0.6$ [S1, unofficial].
*   Grazing boost range: $0.25$ to $0.75$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Self-shadowing:** The normal offset of $0.03$ is critical to prevent self-shadowing artifacts [S1, unofficial].
*   **Metalness Interpolation:** The `mix` function linearly interpolates between non-metal and metal values [S1, unofficial].
*   **Reflection Sampling:** The sky color function must return the environment color for the reflection direction $\mathbf{r}$ [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
