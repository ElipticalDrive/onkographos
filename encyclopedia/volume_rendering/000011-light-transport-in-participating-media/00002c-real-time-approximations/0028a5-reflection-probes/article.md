# Reflection Probes

Also called: Localized Environment Maps, Reflection Proxies

## Overview
Reflection probes approximate incoming radiance by storing sparse environment maps at specific locations in the scene, trading spatial precision for high angular resolution [S1, unofficial]. Unlike precomputed lighting methods that bake static data, reflection probes can be updated dynamically, allowing them to capture dynamic global illumination effects [S1, unofficial]. They differ from voxel-based methods like Voxel Cone Tracing by storing full-resolution directional radiance rather than compressed scattering data, making them suitable for accurate surface reflections but less effective for volumetric scattering integration [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of reflection probe positions (centers), their associated environment maps (cubemaps or spherical projections), and the surface position $\mathbf{p}$ and normal $\mathbf{n}$ of the fragment being shaded.
*   **Output:** The reflected radiance $L_r$ at $\mathbf{p}$.

**Data Structures:**
*   **Reflection Probe:** A struct containing:
    *   `center`: 3D vector $(x, y, z)$ representing the probe's origin.
    *   `radius`: Scalar defining the influence or capture volume.
    *   `environment_map`: Texture array or cubemap storing radiance values.
*   **Environment Map:** A 2D or 3D texture where each texel stores RGB radiance values corresponding to a specific viewing direction.

**Steps:**
1.  **Probe Selection:** Identify the nearest reflection probe to the surface point $\mathbf{p}$. Let this probe's center be $\mathbf{c}$ and its environment map be $E$.
2.  **Direction Calculation:**
    *   Calculate the view direction $\mathbf{v}$ from the surface to the camera.
    *   Calculate the reflection direction $\mathbf{r}$ using the surface normal $\mathbf{n}$:
        $$ \mathbf{r} = 2(\mathbf{v} \cdot \mathbf{n})\mathbf{n} - \mathbf{v} $$
    *   *Note:* In standard implementations, $\mathbf{r}$ is used directly as the lookup direction in the environment map relative to the probe's local coordinate system.
3.  **Lookup:** Sample the environment map $E$ using the direction $\mathbf{r}$ (transformed into the probe's local space). This yields the incoming radiance $L_{in}$.
4.  **Shading:** Compute the reflected radiance using the material's BRDF $f_r$:
    $$ L_r = f_r(\mathbf{v}, \mathbf{r}) \cdot L_{in} $$

**Parameters:**
*   **Update Rate:** How frequently the environment maps are re-rendered. Can be static, per-frame, or temporal.
*   **Resolution:** The resolution of the cubemap faces (e.g., 256x256, 512x512).

**Edge Cases and Pitfalls:**
*   **Distance Deviation:** If an object is far from the probe's center $\mathbf{c}$, the assumption that the environment map is centered at $\mathbf{c}$ becomes invalid, causing reflections to deviate significantly from reality [S1, unofficial].
*   **Occlusion:** Standard environment maps do not account for occlusion between the probe center and the surface. If the probe is inside an object or blocked by geometry, incorrect radiance is sampled.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def shade_with_reflection_probe(surface_point, surface_normal, camera_position, probes):
    # 1. Find nearest probe
    nearest_probe = None
    min_dist = infinity
    for probe in probes:
        dist = distance(surface_point, probe.center)
        if dist < min_dist:
            min_dist = dist
            nearest_probe = probe
    
    if nearest_probe is None:
        return black_color # No probe available

    # 2. Calculate reflection direction in world space
    view_direction = normalize(camera_position - surface_point)
    reflection_direction = reflect(view_direction, surface_normal)
    
    # 3. Transform reflection direction to probe's local space
    # Assuming probe has a rotation matrix R_probe
    local_reflection = transform_direction(reflection_direction, nearest_probe.rotation)
    
    # 4. Sample environment map
    # E is the cubemap texture
    radiance = sample_cubemap(nearest_probe.environment_map, local_reflection)
    
    # 5. Apply BRDF (simplified diffuse/specular split)
    # f_r is the material's BRDF function
    reflected_radiance = nearest_probe.material.brdf(view_direction, local_reflection, radiance)
    
    return reflected_radiance
```

## Subtopics

### Parallax Correction
Parallax correction addresses the deviation error when a surface point is far from the probe center. It assumes incident lighting comes from a sphere with finite size. Instead of using the direct reflection direction, a ray is cast from the surface location through the reflection direction to intersect a light sphere. The direction from the probe center to this intersection point is used as the look direction, effectively fixing the environment map in space relative to the light source geometry [S1, unofficial].

### Reflection Proxies
Reflection proxies generalize reflection probes by allowing any shape (not just spheres centered at the probe) to define the lookup volume. The shape must be ray-intersectable. Accuracy improves as the proxy's shape and size match the surface's local geometry, allowing for more precise spatial approximation than spherical probes [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per fragment for lookup, assuming nearest probe is pre-selected or spatially hashed. Probe update cost is $O(N)$ where $N$ is the number of probes and resolution.
*   **Memory Complexity:** $O(P \cdot R^2)$ where $P$ is the number of probes and $R$ is the cubemap resolution.
*   **Strengths:** High angular resolution; supports dynamic scenes if updated per-frame; simple implementation.
*   **Limitations:** Poor spatial accuracy for objects far from probe centers; no occlusion handling in standard maps; high memory usage for high-resolution cubemaps.
*   **When to use:** For accurate surface reflections in dynamic scenes where volumetric scattering is not the primary concern.
*   **When not to use:** For volumetric scattering integration or when high spatial precision of incident radiance is required across large areas.

## Background
Reflection probes, also known as localized environment maps, were popularized as a real-time alternative to precomputed lighting for capturing dynamic global illumination. The concept of using sparse environment maps to trade spatial precision for angular resolution is described in [S1, unofficial]. Parallax correction was introduced by Brennan [S1, unofficial] to mitigate distance-related errors. The technique is widely used in real-time engines for specular reflections and indirect lighting approximation.

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
