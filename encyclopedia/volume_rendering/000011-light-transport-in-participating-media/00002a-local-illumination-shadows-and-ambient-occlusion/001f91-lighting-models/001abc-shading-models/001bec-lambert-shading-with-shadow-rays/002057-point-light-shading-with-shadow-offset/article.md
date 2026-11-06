# Point Light Shading with Shadow Offset

Also called: Point Lighting with Shadow Rays

## Overview
This method computes direct diffuse illumination from a point light at a surface intersection point while mitigating self-shadowing artifacts (shadow acne) through geometric biasing of the shadow ray origin [S1, S2, unofficial]. It differs from standard shadow ray casting by explicitly offsetting the ray origin along the surface normal based on the dot product of the incoming ray and the normal, ensuring the ray starts slightly above the surface rather than exactly on it [S1, S2, unofficial].

## How it works
**Inputs:** Surface intersection point $\mathbf{p}$, surface normal $\mathbf{n}$, incoming ray direction $\mathbf{d}_{in}$, light position $\mathbf{l}_{pos}$, light radiance $L$, material reflectance $f_r$, and maximum shadow ray distance $D_{max}$ (DRAW-DISTANCE) [S1, S2, unofficial].
**Outputs:** Radiance contribution $L_r$ from the point light [S1, unofficial].

**Data Structures:**
*   **Ray:** Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
*   **Intersection:** Contains point $\mathbf{p}$, normal $\mathbf{n}$, and distance $t$ [S1, S2, unofficial].

**Steps:**
1.  **Compute Light Direction:** Calculate the vector from the surface point to the light source: $\mathbf{v} = \mathbf{l}_{pos} - \mathbf{p}$. Compute the squared distance $d^2 = \|\mathbf{v}\|^2$ and the normalized light direction $\mathbf{l} = \mathbf{v} / \sqrt{d^2}$ [S1, S2, unofficial].
2.  **Compute Cosine Term:** Calculate the dot product between the light direction and the surface normal: $\cos\theta = \mathbf{l} \cdot \mathbf{n}$. Clamp this value to $[0, 1]$ to handle back-facing lights [S1, S2, unofficial].
3.  **Determine Shadow Ray Origin:** To prevent self-intersection, offset the ray origin $\mathbf{o}_{shadow}$ from $\mathbf{p}$ along the normal $\mathbf{n}$. The direction of the offset depends on the sign of the dot product between the incoming ray direction $\mathbf{d}_{in}$ and the normal $\mathbf{n}$ [S1, unofficial].
    *   If $\mathbf{d}_{in} \cdot \mathbf{n} < 0$ (ray coming from above), offset in the direction of $\mathbf{n}$.
    *   Otherwise, offset in the direction of $-\mathbf{n}$ [S1, unofficial].
    *   The offset magnitude is a small bias value, typically $\epsilon = 10^{-9}$ [S1, unofficial].
    *   Implementation note: In raymarching contexts, the offset may be scaled by the step size (e.g., `MARCH-DELTA * 2`) rather than a fixed epsilon [S2, unofficial].
4.  **Trace Shadow Ray:** Cast a ray from $\mathbf{o}_{shadow}$ in direction $\mathbf{l}$ towards the light.
5.  **Determine Occlusion:**
    *   If the shadow ray hits an object at distance $d_{shadow} < D_{max}$, the point is in shadow ($inShadow = true$) [S1, S2, unofficial].
    *   Otherwise, the point is lit ($inShadow = false$) [S1, S2, unofficial].
6.  **Calculate Radiance:** Compute the final radiance using the point light formula [S1, unofficial]:
    $$ L_r = \text{clamp}\left( \frac{L \cdot f_r \cdot \cos\theta \cdot (1 - inShadow)}{d^2 \cdot 4\pi} \right) $$
    *   Note: Source [S2, unofficial] uses a simplified attenuation model where the lightness is multiplied by $0.5$ if in shadow, rather than zeroing it out, and does not explicitly include the $4\pi$ term in the provided pseudocode snippet, suggesting a normalized or simplified lighting model [S2, unofficial]. The standard physical model [S1, unofficial] includes the inverse square law and $4\pi$ factor.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_light_shading_with_shadow_offset(p, n, d_in, l_pos, L, fr, D_max, epsilon=1e-9):
    # 1. Compute light direction and distance
    v = l_pos - p
    d_sq = dot(v, v)
    if d_sq == 0:
        return 0.0
    l = v / sqrt(d_sq)
    
    # 2. Compute cosine term
    cos_theta = max(0.0, dot(l, n))
    
    # 3. Determine shadow ray origin with offset
    # Offset direction based on incoming ray vs normal
    # If ray comes from above (dot < 0), offset along normal
    # Otherwise offset against normal
    if dot(d_in, n) < 0:
        o_shadow = p + n * epsilon
    else:
        o_shadow = p - n * epsilon
        
    # 4. Trace shadow ray (pseudo-call to raymarcher)
    # Returns distance to hit or D_max if no hit
    d_shadow = march_ray(o_shadow, l, D_max)
    
    # 5. Determine occlusion
    in_shadow = d_shadow < D_max
    
    # 6. Calculate radiance
    # Using the formula from [S1, unofficial]
    attenuation = 1.0 if not in_shadow else 0.0
    
    # Note: [S2, unofficial] uses attenuation factor 0.5 instead of 0.0 for shadow
    # and omits 4*pi. We stick to [S1, unofficial] for physical correctness unless specified.
    # If using [S2, unofficial] style: attenuation = 0.5 if in_shadow else 1.0
    
    radiance = (L * fr * cos_theta * attenuation) / (d_sq * 4 * pi)
    
    return radiance
```

**Parameters:**
*   `epsilon`: Geometric bias for shadow ray origin. Value: $10^{-9}$ [S1, unofficial]. In raymarching, this may be `MARCH-DELTA * 2` [S2, unofficial].
*   `fr`: Diffuse material reflectance. Value: $1/\pi$ [S1, unofficial].
*   `D_max`: Maximum distance for the shadow ray. If the ray hits within this distance, the point is in shadow [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Shadow Acne:** Occurs when the shadow ray originates exactly on the surface, causing self-intersection due to floating-point precision errors. Mitigated by the geometric offset [S1, S2, unofficial].
*   **Precision:** The offset must be large enough to prevent self-intersection but small enough to avoid visible gaps ( peter-panning). A fixed epsilon like $10^{-9}$ is standard [S1, unofficial].
*   **Back-facing Lights:** The cosine term is clamped to 0, ensuring no illumination from lights behind the surface [S1, S2, unofficial].

## Subtopics

### Geometric Shadow Bias
A technique to offset shadow ray origins to prevent self-shadowing. It differs from percentage-closer filtering (PCF) by being a geometric solution rather than a sampling-based one. Choose it for simple raymarched scenes where PCF is not available or too expensive [S1, S2, unofficial].

### Point Light Attenuation
The physical model of light falloff over distance. It differs from constant or linear attenuation by using inverse square law. Choose it for physically based rendering [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per light per pixel, assuming the shadow ray trace is $O(1)$ (e.g., via analytical intersection or fixed-step raymarching).
*   **Memory Complexity:** $O(1)$, requiring only local variables for the ray and intersection [S1, S2, unofficial].
*   **Strengths:** Simple to implement; effectively removes shadow acne in raymarched scenes [S1, S2, unofficial].
*   **Limitations:** The fixed offset can cause peter-panning if too large; does not handle soft shadows [S1, S2, unofficial].
*   **When to use:** Real-time raymarched rendering with point lights where shadow acne is a concern [S1, S2, unofficial].
*   **When not to use:** When physically accurate soft shadows are required [S1, unofficial].

## Background
The method originates from practical implementations of raymarched rendering, specifically in the context of Fennel and LÖVE [S2, unofficial] and C++ volume scattering projects [S1, unofficial]. It addresses the common issue of self-shadowing artifacts in raymarching where the shadow ray origin coincides with the surface point [S1, S2, unofficial].

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
[S2] Raymarching with Fennel and LÖVE · Andrey Listopadov, https://andreyor.st/posts/2020-10-15-raymarching-with-fennel-and-love/, fetched 2026-10-30 (unofficial)
