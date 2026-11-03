# Point Light Shading with Shadow Offset


## Overview
This method computes direct illumination from a point light at a surface intersection point, incorporating shadow testing with a geometric offset to prevent self-shadowing artifacts such as shadow acne [S1, unofficial]. It extends basic Lambert shading by introducing a bias term to the shadow ray origin to ensure robust occlusion detection [S1, unofficial].

## How it works
**Inputs:**
- Intersection point $P$ on the surface [S1, unofficial].
- Surface normal $N$ (normalized) [S1, unofficial].
- Incoming camera ray direction $V$ (normalized, pointing towards the surface) [G].
- Light position $L_{pos}$ (world space) [G].
- Light radiance $L$ [S1, unofficial].
- Material reflectance $f_r$ [S1, unofficial].
- Shadow offset bias $\epsilon$ [S1, unofficial].

**Outputs:**
- Reflected radiance $L_r$ at point $P$ [S1, unofficial].

**Data Structures:**
- `Ray`: Struct containing origin $O$ and direction $D$ [G].
- `Intersection`: Struct containing point $P$, normal $N$, and distance $t$ [G].

**Parameters:**
- $\epsilon$: Geometric bias for shadow ray origin. Typical value: $10^{-9}$ [S1, unofficial].
- $f_r$: Diffuse material reflectance. Typical value: $1/\pi$ for ideal diffuse [S1, unofficial].

**Steps:**
1. Compute intersection point $P$ and surface normal $N$ [S1, unofficial].
2. Compute light direction $\vec{L}_{dir} = L_{pos} - P$ and squared distance $d^2 = \|\vec{L}_{dir}\|^2$ [S1, unofficial].
3. Apply a small offset to the shadow ray origin along the normal direction based on the sign of the dot product between the camera ray direction $V$ and the normal $N$ [S1, unofficial].
4. Trace a shadow ray to the light [S1, unofficial].
5. If the shadow ray hits an object closer than the light, mark as in shadow ($\text{inShadow} = \text{true}$) [S1, unofficial].
6. Compute cosine term $\cos\theta$ [S1, unofficial].
7. Calculate radiance using the point light formula [S1, unofficial].
8. Multiply by material reflectance [S1, unofficial].

**Formula:**
$$L_r = \text{clamp}\left( \frac{L \cdot f_r \cdot \cos\theta \cdot (1 - \text{inShadow})}{d^2 \cdot 4\pi} \right)$$
where $L$ is light radiance, $f_r$ is material reflectance, $\cos\theta$ is the clamped dot product of light direction and normal, $\text{inShadow}$ is a boolean, and $d$ is distance to light [S1, unofficial].

**Pitfalls:**
Shadow acne occurs when the shadow ray originates exactly on the surface [S1, unofficial]. It is handled by shifting the origin along the normal by $\epsilon$, with direction determined by the sign of $V \cdot N$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_light_shading_with_shadow_offset(P, N, V, L_pos, L, fr, epsilon):
    # 1. Compute light direction and squared distance
    L_dir = L_pos - P
    d_sq = dot(L_dir, L_dir)
    if d_sq < 1e-12:
        return 0.0
    d = sqrt(d_sq)
    L_dir_norm = L_dir / d

    # 2. Determine shadow ray origin with offset
    # Direction determined by sign of dot product between camera ray (V) and normal (N)
    sign = 1.0 if dot(V, N) > 0.0 else -1.0
    O_shadow = P + epsilon * sign * N

    # 3. Trace shadow ray
    # A step the sources do not give is a comment saying so.
    # hit = trace_scene(shadow_ray, min_t=epsilon, max_t=d)
    in_shadow = False # Default state per standard ray tracing logic [G]

    # 4. Compute cosine term
    cos_theta = max(0.0, dot(N, L_dir_norm))

    # 5. Calculate radiance
    # Formula: L_r = clamp(L * fr * cosθ * (1 - inShadow) / (d² * 4π))
    numerator = L * fr * cos_theta * (1.0 if not in_shadow else 0.0)
    denominator = d_sq * 4.0 * 3.14159265358979323846
    if denominator == 0.0:
        return 0.0
    L_r = numerator / denominator
    return max(0.0, min(1.0, L_r))
```

## Operating characteristics
- **Time Complexity:** $O(1)$ shading calculation plus the cost of one shadow ray trace per point light [G]. The shadow ray trace complexity depends on the scene acceleration structure (e.g., $O(\log N)$ for BVH) [G].
- **Memory Complexity:** $O(1)$ additional memory for local variables [G].
- **Strengths:** Simple to implement; effectively reduces shadow acne artifacts common in basic shadow ray implementations [G].
- **Limitations:** The fixed offset $\epsilon$ may cause "Peter Panning" (shadow detachment) if too large, or fail to prevent acne if too small [G]. It does not account for soft shadows or penumbrae [G].
- **When to use:** Real-time rendering or offline rendering where point lights are used and shadow acne is a visible artifact [G].
- **When not to use:** When soft shadows are required, or when the offset causes noticeable visual artifacts due to large surface curvature or small feature sizes [G].

## Background
This method is a practical extension of basic shadow ray shading, commonly found in ray tracing implementations to handle the numerical precision issues of floating-point arithmetic when testing for self-intersection [G]. The specific implementation details, including the formula and offset logic, are documented in the `VolumeSingleScattering` repository by WangYue [S1, unofficial].

## Sources
[S1] wangyuehi/VolumeSingleScattering, https://github.com/wangyuehi/VolumeSingleScattering, fetched 2026-10-26 (unofficial)
