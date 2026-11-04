# Classical Ray Tracing Integration

Also called: Kajiya and Von Herzen

## Overview
Classical Ray Tracing Integration is a volume rendering technique that computes pixel color by combining background transmission (attenuated by absorption) and uniform in-scattering [S2, S5, unofficial]. It solves the problem of rendering volumetric objects with defined boundaries or integrating geometric visibility directly into volume pipelines [S1, S4, unofficial]. Unlike standard direct volume ray casting which samples uniformly along the entire ray, this method explicitly calculates entry and exit points to limit sampling to the volume interior and applies Beer's law for transmission [S1, S2, unofficial]. It is distinguished from [Alpha Compositing Ray Casting](922-alpha-compositing-ray-casting.md) by its handling of homogeneous spheres via analytical integration, and from [Single Scattering Volume Rendering with Ray Marching](1532-single-scattering-volume-rendering-with-ray-marching.md) by its assumption of uniform scattering color [S2, S3, unofficial].

## How it works
The method operates in two primary modes: analytical integration for homogeneous spheres and ray marching with shading heuristics for general volume data [S2, S3, unofficial]. Both modes rely on the volume rendering integral discretized into a summation over sample points along a ray [Parent principles].

### Inputs and Outputs
*   **Inputs**: Camera ray (origin $O$, direction $D$), volume data (3D grid or sphere parameters), transfer function, shading parameters [S2, S3, unofficial].
*   **Outputs**: Final pixel color $C_{final}$ [S2, S3, unofficial].

### Data Structures
1.  **Sphere Object** (analytical mode):
    *   `center`: `vec3` (e.g., $[0, 0, -4]$) [S2, unofficial].
    *   `radius`: `float` (e.g., $1.0$) [S2, unofficial].
    *   `sigma_a`: `float` (absorption coefficient, e.g., $0.1$) [S2, unofficial].
    *   `scatter`: `vec3` (uniform scatter color, e.g., $[0.8, 0.1, 0.5]$) [S2, unofficial].
2.  **Volume Context** (marching mode):
    *   `volume`: 3D texture or grid [S3, unofficial].
    *   `tfn`: Transfer function mapping scalar value to $(C, \alpha)$ [S3, unofficial].
    *   `params`: Shading mode (`GRADIENT_SHADING` or `SINGLE_SHADE_HEURISTIC`), `shadingScale`, `gradientStep`, lighting parameters [S3, unofficial].

### Algorithm Steps

#### Mode 1: Analytical Integration for Homogeneous Spheres [S2, unofficial]
1.  **Intersection Test**: Compute parametric distances $t_0$ and $t_1$ where the ray enters and leaves the sphere. If no intersection, return `background_color` [S2, unofficial].
2.  **Distance Calculation**: Compute the distance traveled through the sphere as $d = t_1 - t_0$ [S2, unofficial].
3.  **Transmission**: Apply Beer's law to compute the transmission factor $T$:
    $$ T = e^{-\sigma_a \cdot d} $$
    where $\sigma_a$ is the absorption coefficient [S2, unofficial].
4.  **Color Composition**: Compute the final color as the sum of the attenuated background and the uniform scatter:
    $$ C_{final} = C_{background} \cdot T + C_{scatter} \cdot (1 - T) $$
    [S2, unofficial].

#### Mode 2: Ray Marching with Shading Heuristics [S3, unofficial]
1.  **Initialization**:
    *   Initialize `alpha = 0`, `color = 0` [S3, unofficial].
    *   For `SINGLE_SHADE_HEURISTIC`: Initialize `highestOrg = 0`, `highestColor = 0`, `highestAlpha = 0` [S3, unofficial].
2.  **Intersection Check**: If the ray intersects the volume bounds $[t_{start}, t_{end}]$, proceed. Otherwise, return background [S3, unofficial].
3.  **Jitter**: Generate a random jitter value $j \in [0, 1)$ to offset the first sample position along the ray direction [S3, unofficial].
4.  **Ray Marching Loop**: Iterate $t$ from $t_{start}$ to $t_{end}$ with step size `marchingStep`.
    *   **Position**: Calculate position $p = O + t \cdot D$ [S3, unofficial].
    *   **Sample**: Get scalar value $d$ using trilinear interpolation [Parent principles].
    *   **Transfer Function**: Map $d$ to sample color $C_s$ and opacity $\alpha_s$ [S3, unofficial].
    *   **Opacity Correction**: Apply opacity correction based on step size to ensure slice-count invariance [S3, unofficial].
    *   **Gradient Calculation**: Compute gradient $N_o = -\nabla d$ using a central difference operator with `gradientStep` [S3, unofficial]. Transform to world space $N_w$ [S3, unofficial].
    *   **Transmittance**: Calculate current transmittance $tr = 1 - \alpha$ [S3, unofficial].
    *   **Shading Mode**:
        *   **If `GRADIENT_SHADING`**: Transform ray direction to world space. Compute shaded color $C_{shade}$ using lighting model (ambient + directional) with normal $N_w$ [S3, unofficial]. Blend sample color: $C_s = \text{lerp}(\text{shadingScale}, C_s, C_{shade})$ [S3, unofficial].
        *   **If `SINGLE_SHADE_HEURISTIC`**: Check if current sample density is highest: if $\alpha_s \cdot (1 - \alpha) > \text{highestAlpha}$, update `highestOrg`, `highestColor`, and `highestAlpha` [S3, unofficial]. Use unshaded $C_s$ for accumulation [S3, unofficial].
    *   **Accumulation**:
        *   Update color: $C_{new} = C_{old} + (1 - \alpha_{old}) \cdot C_s \cdot \alpha_s$ [S3, unofficial].
        *   Update alpha: $\alpha_{new} = \alpha_{old} + (1 - \alpha_{old}) \cdot \alpha_s$ [S3, unofficial].
    *   **Termination**: If $\alpha \ge 0.99$ (nearly one), break loop [S3, unofficial].
5.  **Final Shading (Heuristic Only)**:
    *   If `highestAlpha > 0`, compute shadow ray transmittance $T_{shadow}$ from `highestOrg` to light direction [S3, unofficial].
    *   Compute final color:
        $$ C_{final} = \text{lerp}(C_{accumulated}, C_{highest} \cdot \alpha_{highest} \cdot T_{shadow}, \text{shadingScale}) $$
        [S3, unofficial].
6.  **Gamma Correction**: Apply gamma 2.2 correction to the final color [Parent principles].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def vec3_add(a, b):
    return (a[0] + b[0], a[1] + b[1], a[2] + b[2])

def vec3_scale(v, s):
    return (v[0] * s, v[1] * s, v[2] * s)

def vec3_pow(v, p):
    return (v[0] ** p, v[1] ** p, v[2] ** p)

def compute_ray_exit(origin, direction, bounds):
    # Slab intersection for axis-aligned box [G]
    t0 = max((bounds[0][i] - origin[i]) / direction[i] if direction[i] != 0 else float('inf') for i in range(3))
    t1 = min((bounds[1][i] - origin[i]) / direction[i] if direction[i] != 0 else float('inf') for i in range(3))
    return max(t0, t1)

def trilinear_interpolate(volume, pos, dims):
    # Standard trilinear interpolation [G]
    vx = pos[0] * (dims[0] - 1)
    vy = pos[1] * (dims[1] - 1)
    vz = pos[2] * (dims[2] - 1)
    x0, y0, z0 = int(math.floor(vx)), int(math.floor(vy)), int(math.floor(vz))
    x1, y1, z1 = x0 + 1, y0 + 1, z0 + 1
    x0 = max(0, min(x0, dims[0]-1))
    x1 = max(0, min(x1, dims[0]-1))
    y0 = max(0, min(y0, dims[1]-1))
    y1 = max(0, min(y1, dims[1]-1))
    z0 = max(0, min(z0, dims[2]-1))
    z1 = max(0, min(z1, dims[2]-1))
    wx = vx - x0
    wy = vy - y0
    wz = vz - z0
    c00 = volume[x0][y0][z0] * (1-wx) + volume[x1][y0][z0] * wx
    c01 = volume[x0][y0][z1] * (1-wx) + volume[x1][y0][z1] * wx
    c10 = volume[x0][y1][z0] * (1-wx) + volume[x1][y1][z0] * wx
    c11 = volume[x0][y1][z1] * (1-wx) + volume[x1][y1][z1] * wx
    r0 = c00 * (1-wy) + c10 * wy
    r1 = c01 * (1-wy) + c11 * wy
    return r0 * (1-wz) + r1 * wz

def classical_ray_tracing_integration(ray_origin, ray_direction, sphere, background_color, volume=None, params=None):
    """
    Renders a pixel using classical ray tracing integration.
    Supports homogeneous spheres (analytical) and general volumes (marching).
    """
    def lerp(a, b, t):
        return a + t * (b - a)

    # --- Mode 1: Homogeneous Sphere (Analytical) ---
    if sphere is not None and volume is None:
        t0, t1 = sphere.intersect(ray_origin, ray_direction)
        if t0 is None:
            return background_color

        distance = t1 - t0
        transmission = math.exp(-sphere.sigma_a * distance)

        final_color = vec3_add(
            vec3_scale(background_color, transmission),
            vec3_scale(sphere.scatter, 1.0 - transmission)
        )
        return vec3_pow(final_color, 1.0 / 2.2)

    # --- Mode 2: General Volume (Ray Marching) ---
    if volume is not None:
        t_start, t_end = volume.intersect_bounds(ray_origin, ray_direction)
        if t_start is None:
            return background_color

        alpha = 0.0
        color = (0.0, 0.0, 0.0)
        highest_org = (0.0, 0.0, 0.0)
        highest_color = (0.0, 0.0, 0.0)
        highest_alpha = 0.0

        jitter = random.random()
        step_size = params.marchingStep if params else 0.01
        t_curr = t_start + jitter * step_size

        while alpha < 0.99 and t_curr < t_end:
            p = vec3_add(ray_origin, vec3_scale(ray_direction, t_curr))
            scalar = trilinear_interpolate(volume.data, p, volume.dims)
            sample_color, sample_alpha = volume.tfn(scalar)

            # Opacity correction for slice-count invariance [G]
            sample_alpha = 1.0 - math.exp(-sample_alpha * step_size)

            grad = volume.sample_gradient(p, scalar, params.gradientStep)
            normal_world = volume.transform_normal(grad)
            transmittance = 1.0 - alpha

            if params.mode == "GRADIENT_SHADING":
                dir_world = volume.transform_vector(ray_direction)
                shaded_color = volume.shade_scivis_light(dir_world, normal_world, sample_color,
                                                         params.mat_gradient_shading,
                                                         params.light_ambient,
                                                         params.light_directional_rgb,
                                                         params.light_directional_dir)
                sample_color = vec3_add(vec3_scale(sample_color, 1.0 - params.shadingScale),
                                        vec3_scale(shaded_color, params.shadingScale))

            elif params.mode == "SINGLE_SHADE_HEURISTIC":
                current_density = transmittance * sample_alpha
                if current_density > highest_alpha:
                    highest_org = p
                    highest_color = sample_color
                    highest_alpha = current_density

            contrib = vec3_scale(sample_color, transmittance * sample_alpha)
            color = vec3_add(color, contrib)
            alpha += transmittance * sample_alpha

            t_curr += step_size

        if params.mode == "SINGLE_SHADE_HEURISTIC" and highest_alpha > 0.0:
            # Shadow ray transmittance computation not provided in sources [Comment]
            shadow_transmittance = 1.0 
            final_shaded_color = vec3_scale(highest_color, highest_alpha * shadow_transmittance)
            color = vec3_add(vec3_scale(color, 1.0 - params.shadingScale),
                             vec3_scale(final_shaded_color, params.shadingScale))

        return vec3_pow(color, 1.0 / 2.2)

    return background_color
```

### Parameters
*   `sigma_a`: Absorption coefficient. Typical value: $0.1$ [S2, unofficial].
*   `scatter`: Uniform scatter color. Typical value: $[0.8, 0.1, 0.5]$ [S2, unofficial].
*   `background_color`: Color returned if no intersection. Typical value: $[0.572, 0.772, 0.921]$ [S2, unofficial].
*   `shadingScale`: Scalar blending unshaded and shaded colors [S3, unofficial].
*   `gradientStep`: Step size for gradient estimation [S3, unofficial].
*   `marchingStep`: Step size for ray marching [S3, unofficial].

### Edge Cases and Pitfalls
*   **Redundant Distance Calculation**: Computing the Euclidean distance between entry and exit points ($\|\bar{p}_1 - \bar{p}_2\|$) is correct but computationally redundant if $t_0$ and $t_1$ are known; use $t_1 - t_0$ [S2, unofficial].
*   **Uniform Scattering Assumption**: The analytical mode assumes uniform scattering color regardless of position or direction; it does not account for directional lighting or varying scattering properties [S2, unofficial].
*   **Heuristic Gradient**: In `SINGLE_SHADE_HEURISTIC`, gradients are not accumulated; only the highest density point is stored for deferred shading [S3, unofficial].
*   **Shadow Ray Cost**: Computing shadow transmittance from the highest density point adds significant cost for the heuristic mode [S3, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of samples along the ray [G]. For analytical spheres, it is $O(1)$ per pixel [S2, unofficial]. For marching, it depends on volume density and step size [S3, unofficial].
*   **Memory Complexity**: $O(1)$ for analytical mode. $O(V)$ for volume data storage in marching mode [G].
*   **Performance**: Analytical integration is extremely fast and exact for homogeneous spheres [S2, unofficial]. Ray marching handles complex scalar fields and supports deferred shading heuristics [S3, unofficial].
*   **Strengths**: Exact solution for homogeneous volumes; integrates geometric visibility; supports shading heuristics for complex data [S1, S2, S3, S4, unofficial].
*   **Limitations**: Analytical mode is limited to simple shapes; uniform scattering assumption is unrealistic for many volumes; heuristic shading may produce artifacts if the highest density point is not representative [S2, S3, unofficial].
*   **When to use**: Ideal for rendering simple volumetric objects (e.g., clouds, smoke balls) or when integrating geometric surfaces with volume data [S1, S3, unofficial]. Avoid for highly anisotropic or spatially varying scattering volumes [S2, unofficial].

## Background
The method originates from early volume rendering techniques that combined ray tracing with volume data [S1, unofficial]. The integration of classical ray tracing into volume rendering pipelines was proposed to handle visibility and geometric merging, such as superimposing radiation treatment planning isodose surfaces over CT data [S4, unofficial]. The heuristic shading approach was developed to balance performance and visual quality in scientific visualization [S3, unofficial]. Recent applications include integrating ray tracing for interreflections in 3D Gaussian Splatting [S7, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] An Introduction to Volume Rendering, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/intro-volume-rendering.html, fetched 2026-10-25 (unofficial)
[S3] VIDILabs/instantvnr, https://github.com/VIDILabs/instantvnr, fetched 2026-10-26 (unofficial)
[S4] DTIC ADA193649: Rendering of Surfaces from Volume Data,, https://archive.org/details/DTIC_ADA193649, fetched 2026-10-26 (unofficial)
[S5] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
[S6] Volume and Isosurface Rendering with GPU‐Accelerated Cell Projection, https://www.researchgate.net/publication/227523087_Volume_and_Isosurface_Rendering_with_GPU-Accelerated_Cell_Projection, fetched 2026-10-27 (unofficial)
[S7] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
