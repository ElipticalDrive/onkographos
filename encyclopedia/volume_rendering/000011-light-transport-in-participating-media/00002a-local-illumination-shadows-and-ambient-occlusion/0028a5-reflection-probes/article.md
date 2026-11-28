# Reflection Probes

Also called: Localized Environment Maps

## Overview
Reflection probes approximate incoming radiance by storing sparse environment maps at specific locations in the scene, trading spatial precision for increased angular resolution [S1, unofficial]. Unlike precomputed lighting methods that bake static data, reflection probes can be updated dynamically to capture dynamic global illumination [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of reflection probe positions (centers), their associated environment maps, and the surface position $\mathbf{p}$ and normal $\mathbf{n}$ of the fragment being shaded [G].
*   **Output:** The reflected radiance $L_r$ at $\mathbf{p}$ [G].

**Data Structures:**
*   **Reflection Probe:** A struct containing `center` (3D vector), `rotation` (3x3 matrix), and `environment_map` (cubemap or texture array) [G].
*   **Environment Map:** A texture where each texel stores RGB radiance values corresponding to a specific viewing direction [G].

**Steps:**
1.  **Probe Selection:** Identify the nearest reflection probe to the surface point $\mathbf{p}$ [G].
2.  **Direction Calculation:** Calculate the view direction $\mathbf{v}$ from the surface to the camera [G]. Calculate the reflection direction $\mathbf{r}$ using the surface normal $\mathbf{n}$:
    $$ \mathbf{r} = 2(\mathbf{v} \cdot \mathbf{n})\mathbf{n} - \mathbf{v} $$
    [G]
3.  **Lookup:** Transform $\mathbf{r}$ into the probe's local coordinate system and sample the environment map $E$ [G]. This yields the incoming radiance $L_{in}$ [G].
4.  **Shading:** Compute the reflected radiance using the material's BRDF $f_r$:
    $$ L_r = f_r(\mathbf{v}, \mathbf{r}) \cdot L_{in} $$
    [G]

**Parameters:**
*   **Update Rate:** How frequently the environment maps are re-rendered [G]. Can be static, per-frame, or temporal [S1, unofficial].
*   **Resolution:** The resolution of the cubemap faces. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Distance Deviation:** If an object is far from the environment map's center, reflections deviate significantly from reality [S1, unofficial].
*   **Occlusion:** Standard environment maps do not account for occlusion between the probe center and the surface. If the probe is inside an object or blocked by geometry, incorrect radiance is sampled [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def normalize(v):
    mag = (v[0]**2 + v[1]**2 + v[2]**2)**0.5
    return (v[0]/mag, v[1]/mag, v[2]/mag)

def shade_with_reflection_probe(surface_point, surface_normal, camera_position, probes):
    nearest_probe = None
    min_dist = float('inf')
    for probe in probes:
        dx = surface_point[0] - probe.center[0]
        dy = surface_point[1] - probe.center[1]
        dz = surface_point[2] - probe.center[2]
        dist = (dx*dx + dy*dy + dz*dz)**0.5
        if dist < min_dist:
            min_dist = dist
            nearest_probe = probe

    if nearest_probe is None:
        return (0.0, 0.0, 0.0)

    view_direction = normalize((camera_position[0] - surface_point[0],
                                camera_position[1] - surface_point[1],
                                camera_position[2] - surface_point[2]))
    dot = (view_direction[0] * surface_normal[0] +
           view_direction[1] * surface_normal[1] +
           view_direction[2] * surface_normal[2])
    reflection_direction = (2.0 * dot * surface_normal[0] - view_direction[0],
                            2.0 * dot * surface_normal[1] - view_direction[1],
                            2.0 * dot * surface_normal[2] - view_direction[2])

    R = nearest_probe.rotation
    local_reflection = (
        R[0][0]*reflection_direction[0] + R[0][1]*reflection_direction[1] + R[0][2]*reflection_direction[2],
        R[1][0]*reflection_direction[0] + R[1][1]*reflection_direction[1] + R[1][2]*reflection_direction[2],
        R[2][0]*reflection_direction[0] + R[2][1]*reflection_direction[1] + R[2][2]*reflection_direction[2]
    )

    # Not given in the sources: standard cubemap face selection and UV interpolation
    radiance = nearest_probe.environment_map.lookup(local_reflection)

    # Not given in the sources: standard material BRDF evaluation
    return radiance
```

## Subtopics

### Parallax Correction
Parallax correction addresses the deviation error when a surface point is far from the probe center [S1, unofficial]. It assumes incident lighting comes from a sphere with finite size [S1, unofficial]. Instead of using the direct reflection direction, a ray is cast from the surface location through the reflection direction to intersect a light sphere [S1, unofficial]. The direction from the probe center to this intersection point is used as the look direction, effectively fixing the environment map in space relative to the light source geometry [S1, unofficial]. Choose this variant when probe-surface distance is large relative to the probe radius [G].

### Planar Reflections
Planar reflections reuse the regular scene representation by rerendering it from a different perspective and storing the result in a texture for the final render pass [S1, unofficial]. They differ from localized environment maps by providing exact geometric reflections for flat surfaces rather than approximating them with spherical sampling [S1, unofficial]. Choose this variant for mirror-like planar surfaces [G]. Pitfall: requires a separate render pass for each mirror surface, which becomes prohibitively costly, especially when dealing with recursive reflections [S1, unofficial].

### Reflection Proxies
Reflection proxies generalize reflection probes by allowing any ray-intersectable shape (not just spheres centered at the probe) to define the lookup volume [S1, unofficial]. They differ from standard probes by matching the proxy's shape and size to the surface's local geometry, yielding higher spatial accuracy [S1, unofficial]. Choose this variant when surface curvature or geometry deviates significantly from spherical symmetry [G].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [G] O(1) per fragment for lookup, assuming nearest probe is pre-selected or spatially hashed. Probe update cost is O(N) where N is the number of probes and resolution [G].
*   **Memory Complexity:** Not given in the sources. [G] O(P * R^2) where P is the number of probes and R is the cubemap resolution.
*   **Strengths:** High angular resolution; supports dynamic scenes if updated per-frame [S1, unofficial].
*   **Limitations:** Poor spatial accuracy for objects far from probe centers; no occlusion handling in standard maps [S1, unofficial].
*   **When to use:** For accurate surface reflections in dynamic scenes where volumetric scattering is not the primary concern [G].
*   **When not to use:** For volumetric scattering integration or when high spatial precision of incident radiance is required across large areas [G].

## Background
Reflection probes, also known as localized environment maps, were introduced as a real-time alternative to precomputed lighting for capturing dynamic global illumination [S1, unofficial]. The technique trades spatial precision for angular resolution by sparsely distributing environment maps throughout the scene [S1, unofficial]. Parallax correction was introduced by Brennan to mitigate distance-related errors by assuming incident lighting originates from a finite sphere [S1, unofficial]. The method is widely used in real-time engines for specular reflections and indirect lighting approximation [G].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
