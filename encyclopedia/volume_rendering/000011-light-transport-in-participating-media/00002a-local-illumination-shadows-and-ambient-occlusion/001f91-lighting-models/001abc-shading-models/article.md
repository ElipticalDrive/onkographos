# Shading models

Also called: Diffuse and Specular Shading for Volume Rendering

## Overview
Shading models for volume rendering enhance the perception of 3D structure and surface orientation in scalar field data by applying local illumination heuristics [S1, unofficial]. Unlike physically based light transport methods that solve the radiative transfer equation, this method modulates the existing volume samples or pixel colors using diffuse and specular terms derived from the data gradient [S1, unofficial]. It distinguishes itself from frequency domain methods by operating directly in the spatial domain using explicit vector math [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Scalar volume data $V$, transfer function $T$, camera position $C$, light position $L$, and shading parameters [S1, unofficial].
*   **Output:** Final color $C_{final}$ for each pixel [G].

**Data Structures:**
The method operates on ray-region intersections [S1, unofficial]. For a ray passing through a data cell or region, the following values are computed at the entry (front) and exit (back) points:
*   $d_{entry}, d_{exit}$: Data values (scalar field samples) at entry and exit.
*   $\mathbf{g}_{entry}, \mathbf{g}_{exit}$: Gradient vectors at entry and exit.
*   $|\mathbf{g}_{entry}|, |\mathbf{g}_{exit}|$: Gradient magnitudes.

**Parameters:**
*   $d_{min}, d_{max}$: Shading range limits. Data values outside $[d_{min}, d_{max}]$ are not shaded [S1, unofficial].
*   $g_{min}$: Minimum gradient magnitude threshold. Gradients with magnitude $< g_{min}$ are not shaded [S1, unofficial].
*   $k_d$: Amount of diffuse shading.
*   $k_s$: Amount of specular shading.
*   $\alpha$: Specular shininess exponent.
*   $\beta$: Opacity of the shading contribution.
*   $C_{diffuse\_base}$: Diffuse reflection color, assumed to be $T(d)$, the color from the transfer function at the shaded data value [S1, unofficial].
*   $C_{specular}$: Specular reflection color, assumed to be white $(1, 1, 1)$ [S1, unofficial].

**Steps:**
1.  **Gradient Calculation:** Compute the gradient $\mathbf{g}$ at the intersection points. The source uses the derivative of the trilinear interpolation function as the gradient [S1, unofficial]. Note that gradients at shared cell faces may not be continuous [S1, unofficial].
2.  **Shading Eligibility Check:** Determine if shading should be applied to the ray segment. Shading is applied if:
    *   Either $d_{entry}$ or $d_{exit}$ is within $[d_{min}, d_{max}]$ [S1, unofficial].
    *   AND either $|\mathbf{g}_{entry}|$ or $|\mathbf{g}_{exit}|$ is $\ge g_{min}$ [S1, unofficial].
    If not eligible, the shading contribution is zero [S1, unofficial].
3.  **Average Data Value:** Compute the data value $d_{avg}$ used for diffuse reflectivity:
    $$ d_{avg} = \frac{d_{entry} + d_{exit}}{2} $$
4.  **Average Gradient:** Compute the gradient vector $\mathbf{g}_{avg}$ used for the normal approximation:
    *   If both $|\mathbf{g}_{entry}|$ and $|\mathbf{g}_{exit}|$ are within valid ranges (i.e., $\ge g_{min}$ and corresponding data values in range):
        $$ \mathbf{g}_{avg} = \frac{\mathbf{g}_{entry} + \mathbf{g}_{exit}}{2} $$
    *   If one gradient magnitude is out of range (or its corresponding data value is out of range), use the gradient that is within range:
        $$ \mathbf{g}_{avg} = \begin{cases} \mathbf{g}_{entry} & \text{if } \mathbf{g}_{entry} \text{ is valid} \\ \mathbf{g}_{exit} & \text{if } \mathbf{g}_{exit} \text{ is valid} \end{cases} $$
    *   [G] If neither is valid, shading is skipped (handled in Step 2).
5.  **Normal Normalization:** Compute the surface normal approximation $\mathbf{n}$:
    $$ \mathbf{n} = \frac{\mathbf{g}_{avg}}{|\mathbf{g}_{avg}|} $$
    [G] If $|\mathbf{g}_{avg}| \approx 0$, set $\mathbf{n} = (0,0,0)$ and skip shading.
6.  **Lighting Vectors:**
    *   Light direction $\mathbf{l}$: Vector from hit point $P$ to light $L$, normalized [G].
    *   View direction $\mathbf{v}$: Vector from hit point $P$ to camera $C$, normalized [G].
    *   Half vector $\mathbf{h}$: $\mathbf{h} = \frac{\mathbf{l} + \mathbf{v}}{|\mathbf{l} + \mathbf{v}|}$ [G].
7.  **Diffuse Term:** Compute diffuse factor $D$:
    $$ D = \max(0.0, \mathbf{n} \cdot \mathbf{l}) $$
    Diffuse color $C_{diff} = C_{diffuse\_base} \cdot D$ [S1, unofficial].
8.  **Specular Term:** Compute specular factor $S$:
    $$ S = \max(0.0, \mathbf{n} \cdot \mathbf{h})^\alpha $$
    Specular color $C_{spec} = C_{specular} \cdot S$ [S1, unofficial].
9.  **Combine Shading:** Compute the total shading color $C_{shade}$:
    $$ C_{shade} = k_d C_{diff} + k_s C_{spec} $$
10. **Final Color:** Blend the shading contribution with the underlying volume color $C_{vol}$ (from ray casting) using opacity $\beta$:
    $$ C_{final} = C_{vol} + \beta C_{shade} $$
    [G] Clamp $C_{final}$ to $[0, 1]$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_shading(d_entry, d_exit, g_entry, g_exit, d_min, d_max, g_min, 
                    k_d, k_s, alpha, beta, C_diffuse_base, C_specular, 
                    hit_point, light_pos, camera_pos):
    """
    Computes shading contribution for a ray segment through a volume cell.
    """
    def norm(v):
        return (v[0]**2 + v[1]**2 + v[2]**2)**0.5
    def dot(a, b):
        return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]
    def normalize(v):
        n = norm(v)
        if n < 1e-6:
            return (0.0, 0.0, 0.0)
        return (v[0]/n, v[1]/n, v[2]/n)

    def is_valid(d_val, g_val):
        if d_val < d_min or d_val > d_max:
            return False
        if norm(g_val) < g_min:
            return False
        return True

    valid_entry = is_valid(d_entry, g_entry)
    valid_exit = is_valid(d_exit, g_exit)

    data_in_range = (d_entry >= d_min and d_entry <= d_max) or (d_exit >= d_min and d_exit <= d_max)
    grad_above_thresh = (norm(g_entry) >= g_min) or (norm(g_exit) >= g_min)
    
    if not (data_in_range and grad_above_thresh):
        return (0.0, 0.0, 0.0)

    d_avg = (d_entry + d_exit) / 2.0
    
    g_avg = (0.0, 0.0, 0.0)
    if valid_entry and valid_exit:
        g_avg = ((g_entry[0] + g_exit[0]) / 2.0, (g_entry[1] + g_exit[1]) / 2.0, (g_entry[2] + g_exit[2]) / 2.0)
    elif valid_entry:
        g_avg = g_entry
    elif valid_exit:
        g_avg = g_exit
    else:
        return (0.0, 0.0, 0.0)

    g_norm = norm(g_avg)
    if g_norm < 1e-6:
        return (0.0, 0.0, 0.0)
    n = normalize(g_avg)

    l = normalize(hit_point - light_pos)
    v = normalize(camera_pos - hit_point)
    h = normalize(l + v)

    diff_factor = max(0.0, dot(n, l))
    C_diff = (C_diffuse_base[0] * diff_factor, C_diffuse_base[1] * diff_factor, C_diffuse_base[2] * diff_factor)

    spec_factor = max(0.0, dot(n, h))
    spec_factor = spec_factor ** alpha
    C_spec = (C_specular[0] * spec_factor, C_specular[1] * spec_factor, C_specular[2] * spec_factor)

    C_shade = (
        k_d * C_diff[0] + k_s * C_spec[0],
        k_d * C_diff[1] + k_s * C_spec[1],
        k_d * C_diff[2] + k_s * C_spec[2]
    )
    
    return C_shade
```

## Subtopics

### [Frequency Domain Directional Shading](852-frequency-domain-directional-shading.md)
Frequency Domain Directional Shading is a local illumination technique for volume rendering that computes diffuse shading entirely in the frequency domain, avoiding spatial domain differentiation. It solves the problem of applying directional lighting cues to volume data without the computational cost or nonlinearity issues of spatial domain methods. Unlike [Simplified Phong Lighting Model](885-simplified-phong-lighting-model.md) and [Lambert Shading with Shadow Rays](1133-lambert-shading-with-shadow-rays.md), which operate in the spatial domain or require shadow rays, this method leverages the property that spatial differentiation corresponds to multiplication by a linear ramp in the frequency domain. It differs from [Gradient-based Shading with Phase Functions](1020-gradient-based-shading-with-phase-functions.md) by using a hemispherical light source model to maintain linearity, enabling efficient frequency domain computation.

### [Simplified Phong Lighting Model](885-simplified-phong-lighting-model.md)
The Simplified Phong Lighting Model is a local illumination technique adapted for volume rendering to enhance the perception of 3D structure by simulating surface-like shading on scalar field data. It solves the problem of flat, unlit volume visualization by calculating diffuse and specular contributions based on the gradient of the volume data, which serves as an approximation of the surface normal. Unlike physically based light transport methods, it is a heuristic shading model that modulates the existing pixel color or volume samples to create highlights and shadows, distinguishing itself from its siblings by typically relying on a single light source and explicit vector math rather than precomputed spectral expansions or complex phase functions.

### [Gradient-based Shading with Phase Functions](1020-gradient-based-shading-with-phase-functions.md)
Gradient-based shading with phase functions adds illumination terms to the volume rendering integral to simulate single-scattering effects or computes shading for implicit iso-surfaces, thereby providing a sense of depth that absorption and emission alone cannot achieve. It distinguishes itself from sibling methods by explicitly deriving surface normals from the scalar field gradient and applying phase functions (such as Lambert's law or Henyey-Greenstein) to model light interaction, rather than relying solely on frequency domain approximations or simplified Phong models.

### [Lambert Shading with Shadow Rays](1133-lambert-shading-with-shadow-rays.md)
A shading model that computes diffuse illumination using the Lambertian reflectance model, enhanced with shadow rays to detect occlusion from light sources, improving the perception of surface shape and depth.

### [Volume Shading Model](2991-volume-shading-model.md)
Computes final fragment color by combining indirect diffuse, indirect specular, emissivity, transparency/refraction, and direct light contributions, with shadow blending and gamma correction.

### Fire Shading Model
A shading model that combines Lambertian diffuse lighting, ambient light, and rim lighting, modulated by a fire-colored palette driven by animated fBM noise [S2, unofficial]. It computes the surface normal $N$ using `distance_field_normal(hit)`, light direction $L$ from hit point to light, and view direction $V$ from hit point to camera [S2, unofficial]. The Lambertian diffuse factor is `max(0.0, dot(N, L))` and the rim light factor is `pow(max(0.0, 1.0 - max(0.0, dot(N, V))), 2.0)` [S2, unofficial]. The total light intensity is `ambient + 0.9 * lambert + 0.4 * rim` [S2, unofficial]. The fire color value `fval` is evaluated via fBM at the hit point with time offset, mapped to a fire palette, and multiplied by the light intensity to produce the final color [S2, unofficial]. Parameter `ambient` is a constant ambient light factor [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per shading sample, as it involves basic vector arithmetic and trigonometric functions [G]. The cost is added to the ray casting step [G].
*   **Memory Complexity:** $O(1)$ additional memory per ray, storing intersection points and gradients [G].
*   **Performance:** Shading adds computational overhead to ray casting but is significantly cheaper than full global illumination or path tracing [G]. It helps considerably in bringing out features and establishing orientations [S1, unofficial].
*   **Strengths:** Simple to implement; enhances visual clarity of internal structures; parallelizable (each ray is independent) [S1, unofficial].
*   **Limitations:** Heuristic; not physically accurate; gradient discontinuities at cell boundaries may cause artifacts [S1, unofficial]; sensitive to parameter tuning ($d_{min}, d_{max}, g_{min}$).
*   **When to use:** When visual clarity and structural perception are prioritized over physical accuracy; for real-time or interactive volume visualization [G].
*   **When not to use:** When physically based rendering is required; when accurate light transport (e.g., multiple scattering) is needed [G].

## Background
The method was implemented within a ray casting volume renderer described in a technical report from UCSC [S1, unofficial]. It was part of a hierarchical visualization software suite that compared ray casting with 3D texture mapping and coherent projection [S1, unofficial]. The shading model was designed to address the flat appearance of unshaded volume data, particularly in medical imaging contexts like CT head scans [S1, unofficial]. The use of trilinear derivatives for gradients was chosen for availability, despite potential discontinuities [S1, unofficial].

## Sources
[S1] [PDF] Hierarchically Accelerated Ray Casting for Volume Rendering with ..., https://tr.soe.ucsc.edu/sites/default/files/technical-reports/UCSC-CRL-95-31.pdf, fetched 2026-10-27 (unofficial)
[S2] OmerNYU/Kaboom_Diffusion, https://github.com/OmerNYU/Kaboom_Diffusion, fetched 2026-10-26 (unofficial)
