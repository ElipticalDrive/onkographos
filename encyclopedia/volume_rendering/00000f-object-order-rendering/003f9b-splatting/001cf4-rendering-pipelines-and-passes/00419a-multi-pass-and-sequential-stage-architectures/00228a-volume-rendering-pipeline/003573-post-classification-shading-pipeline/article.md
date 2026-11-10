# Post-classification Shading Pipeline

Also called: Post-DVRI, Post-classification DVR, Post-shaded DVRI

## Overview
The Post-classification Shading Pipeline (also known as Post-DVRI) is a volume rendering method where the original density volume is interpolated, sample values are classified via transfer functions to yield material, opacity, and color, and shading follows immediately after classification [S1]. It is distinct from the Pre-classification Shading Pipeline primarily in its handling of partial voluming and its susceptibility to blurring at high image resolutions [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar density volume $f$, ray direction $r$, ray origin $x$, transfer functions for color $C_\lambda$ and opacity $\alpha$, and step size $\Delta s$ [G].
*   **Output:** The final image intensity $I_\lambda(x,r)$ for each pixel [S1].

**Data Structures:**
*   **Density Volume:** A 3D grid of scalar values $f(x,y,z)$ [G].
*   **Transfer Functions:** Lookup tables mapping density values to color $C_\lambda$ and opacity $\alpha$ [S1].

**Numbered Steps:**
1.  **Ray Casting:** For each pixel, cast a ray through the volume [G].
2.  **Interpolation:** Sample the raw density volume $f$ at intervals $\Delta s$ along the ray. Let $s_i$ be the position of the $i$-th sample, where $i$ ranges from $0$ to $L/\Delta s$ ($L$ is the ray length within the volume). The density value at this position is $f(s_i)$, obtained by interpolating the grid values (e.g., trilinear interpolation) [S1].
3.  **Classification:** Apply the transfer functions to the interpolated density value $f(s_i)$ to determine the color $C_\lambda(f(s_i))$ and opacity $\alpha(f(s_i))$ for that sample [S1].
4.  **Shading:** Compute the gradient of the density grid at the sample location to estimate the surface normal. Use this normal for shading calculations (e.g., Phong or Blinn-Phong model) to determine the final color contribution. This shading step occurs *after* classification but *before* compositing [S1].
5.  **Compositing:** Composite the shaded samples along the ray using front-to-back alpha blending [S1].

**Formula:**
The final intensity $I_\lambda(x,r)$ is computed as [S1]:

$$
I_\lambda(x,r) = \sum_{i=0}^{L/\Delta s} C_\lambda(f(s_i)) \alpha(f(s_i)) \prod_{j=0}^{i-1} (1 - \alpha(f(s_j)))
$$

Where:
*   $L$ is the length of the ray $r$ [S1].
*   $\Delta s$ is the step size [G].
*   $s_i$ is the position of the $i$-th sample along the ray [S1].
*   $f(s_i)$ is the interpolated density value at $s_i$ [S1].
*   $C_\lambda(f(s_i))$ is the color (wavelength $\lambda$) determined by the transfer function applied to $f(s_i)$ [S1].
*   $\alpha(f(s_i))$ is the opacity determined by the transfer function applied to $f(s_i)$ [S1].
*   The product term $\prod_{j=0}^{i-1} (1 - \alpha(f(s_j)))$ represents the transmittance from the ray origin to sample $i$ [S1].

**Parameters:**
*   **Step Size ($\Delta s$):** Determines the sampling rate [G].
*   **Transfer Functions:** Typically implemented as lookup tables [S1].

**Edge Cases and Pitfalls:**
*   **Partial Volume Effect:** Interpolated density values may fall between distinct material densities, resulting in a classification that yields a material not actually present at the sample location. This leads to "false colors" in the final image [S1].
*   **Staircasing Artifacts:** To avoid false colors, prior segmentation can be used. However, this introduces high-frequency discontinuities, leading to severe staircasing artifacts [S1].
*   **Blurriness:** Unlike pre-classification methods, this approach avoids blurriness at high image resolutions (zoomed viewing or wide perspectives) because fine details in the transfer functions are expressed in the final image [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def post_classification_shading_pipeline(volume, volume_bounds, ray_origin, ray_direction, transfer_functions, step_size, light_positions, view_direction):
    # 1. Normalize direction and intersect with volume bounding box
    ray_direction = normalize(ray_direction)
    t_min, t_max = ray_box_intersection(ray_origin, ray_direction, volume_bounds)
    if t_max <= t_min:
        return [0.0, 0.0, 0.0]
    L = t_max - t_min
    num_samples = int(L / step_size)

    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_opacity = 0.0

    for i in range(num_samples):
        t = t_min + i * step_size
        sample_pos = ray_origin + t * ray_direction

        # 2. Interpolate raw volume densities
        density = trilinear_interpolate(volume, sample_pos, volume_bounds)

        # 3. Classify resulting sample values via transfer functions
        color = transfer_functions.get_color(density)
        opacity = transfer_functions.get_opacity(density)

        # 4. Compute gradients from the density grid for shading
        grad_f = compute_gradient(volume, sample_pos, volume_bounds)
        normal = normalize(grad_f)

        # 5. Shading follows immediately after classification
        shaded_color = [0.0, 0.0, 0.0]
        for light_pos in light_positions:
            light_dir = normalize(light_pos - sample_pos)
            diffuse = max(0.0, dot(normal, light_dir))
            shaded_color = [shaded_color[k] + color[k] * diffuse for k in range(3)]

        # 6. Composite
        transmittance = 1.0 - accumulated_opacity
        contribution = [shaded_color[k] * opacity * transmittance for k in range(3)]
        accumulated_color = [accumulated_color[k] + contribution[k] for k in range(3)]
        accumulated_opacity += opacity * transmittance

        if accumulated_opacity >= 1.0:
            break

    return accumulated_color

def ray_box_intersection(origin, direction, bounds):
    t_min, t_max = 0.0, float('inf')
    for i in range(3):
        if direction[i] == 0:
            if origin[i] < bounds[i] or origin[i] > bounds[i+3]:
                return 0.0, 0.0
        else:
            t1 = (bounds[i] - origin[i]) / direction[i]
            t2 = (bounds[i+3] - origin[i]) / direction[i]
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
    return t_min, t_max

def trilinear_interpolate(volume, pos, bounds):
    # Standard trilinear interpolation [G]
    # Maps continuous pos to discrete grid indices and interpolates
    # Implementation details omitted for brevity, but standard [G]
    # For self-containment, a minimal version:
    min_b = bounds[:3]
    max_b = bounds[3:]
    res = [max_b[i] - min_b[i] for i in range(3)]
    idx = [(pos[i] - min_b[i]) / res[i] * (len(volume[i])-1) for i in range(3)]
    idx = [max(0, min(len(volume[i])-2, int(idx[i]))) for i in range(3)]
    frac = [pos[i] - (min_b[i] + idx[i] * res[i] / (len(volume[i])-1)) for i in range(3)]
    v000 = volume[idx[0]][idx[1]][idx[2]]
    v100 = volume[idx[0]+1][idx[1]][idx[2]]
    v010 = volume[idx[0]][idx[1]+1][idx[2]]
    v110 = volume[idx[0]+1][idx[1]+1][idx[2]]
    v001 = volume[idx[0]][idx[1]][idx[2]+1]
    v101 = volume[idx[0]+1][idx[1]][idx[2]+1]
    v011 = volume[idx[0]][idx[1]+1][idx[2]+1]
    v111 = volume[idx[0]+1][idx[1]+1][idx[2]+1]
    c00 = v000 + frac[0]*(v100-v000)
    c01 = v001 + frac[0]*(v101-v001)
    c10 = v010 + frac[0]*(v110-v010)
    c11 = v011 + frac[0]*(v111-v011)
    c0 = c00 + frac[1]*(c10-c00)
    c1 = c01 + frac[1]*(c11-c01)
    return c0 + frac[2]*(c1-c0)

def compute_gradient(volume, pos, bounds):
    # Central differences [G]
    eps = 1e-4
    grad = [0.0, 0.0, 0.0]
    for i in range(3):
        p_plus = list(pos); p_plus[i] += eps
        p_minus = list(pos); p_minus[i] -= eps
        grad[i] = (trilinear_interpolate(volume, p_plus, bounds) - trilinear_interpolate(volume, p_minus, bounds)) / (2*eps)
    return grad

def normalize(v):
    mag = (v[0]**2 + v[1]**2 + v[2]**2)**0.5
    return [v[i]/mag for i in range(3)]

def dot(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]
```

## Subtopics

### [Pre-classification Shading Pipeline](7803-pre-classification-shading-pipeline.md)
The Pre-classification Shading Pipeline (also known as Pre-DVRI) is a volume rendering method where voxel values are mapped to optical properties (color and opacity) and shaded *before* ray sample interpolation occurs. Unlike the Post-classification Shading Pipeline, which interpolates raw density values first, this approach classifies and shades the discrete grid samples, then composites them along the ray. It is distinct from post-classification primarily in its handling of partial voluming and its susceptibility to blurring at high image resolutions.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples per ray and the number of rays (pixels) [G].
*   **Memory Complexity:** Linear with respect to the volume size for storage [G].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Performance scales with image resolution and volume size [G].
*   **Strengths:** Avoids blurriness at high image resolutions (zoomed viewing or wide perspectives) because fine details in transfer functions are expressed in the final image [S1].
*   **Limitations:** Susceptible to false colors due to the partial volume effect, where interpolated densities are classified as materials not present at the sample location [S1]. Prior segmentation to avoid false colors can introduce severe staircasing artifacts [S1].
*   **When to use:** When high-resolution detail preservation is critical and partial volume artifacts are acceptable or can be mitigated [G].
*   **When not:** When partial volume effects cause unacceptable false colors and segmentation is not feasible or introduces too much noise [G].

## Background
The Post-classification Shading Pipeline, also termed Post-DVRI (post-shaded DVRI), was described in the context of comparing popular volume rendering algorithms [S1]. It is one of the two common optical models for volume rendering, depending on the order of classification/interpolation components [S1]. The algorithms differ mostly in the way interpolation and sampling is performed and also how interpolations are avoided in less relevant or irrelevant volume regions [S1].

## Sources
[S1] A Practical Evaluation of Popular Volume Rendering ..., https://courses.cs.duke.edu/spring03/cps296.8/papers/PracticalEvaluationofVolumeRenderingAlgorithms.pdf, fetched 2026-10-29
