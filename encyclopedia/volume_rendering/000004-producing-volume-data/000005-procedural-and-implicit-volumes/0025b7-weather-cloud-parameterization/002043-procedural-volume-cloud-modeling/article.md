# Procedural Volume Cloud Modeling

Also called: Cloud Density Field Generation, Density Sampling and Weather

## Overview
Procedural Volume Cloud Modeling constructs the volumetric density field of clouds by evaluating physical and visual parameters at arbitrary world-space coordinates, primarily using 3D noise textures and weather maps [S1, S2, unofficial]. It solves the problem of generating realistic, large-scale cloud formations in real-time by replacing pre-baked static volumes with dynamic, parameter-driven density evaluation [S2, unofficial]. It differs from [Volumetric Cloud Raymarching](1248-volumetric-cloud-raymarching.md) by focusing on the generation of the density scalar rather than the integration of light through it, and from [Cloud Density Sampling](4806-cloud-density-sampling.md) by explicitly combining height gradients, weather map attributes, and multi-octave noise erosion in a single procedural evaluation step [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** World-space position $P$, time $t$, wind vectors, and global parameters defined in the parent [Weather & Cloud Parameterization](3609-weather-cloud-parameterization.md) [S1, unofficial].
*   **Outputs:** A scalar density value $\rho$ at $P$ [S1, unofficial].

**Data Structures:**
1.  **Cloud Map:** A 2D texture with 4 channels (RGBA) [S2, unofficial].
    *   R: Cloud Coverage
    *   G: Cloud Type
    *   B: Cloud Erosion
    *   A: Cloud Wetness
2.  **Cloud Lut:** A texture defining distribution of cloud types and heights [S2, unofficial].
    *   R: Density
    *   G: Erosion
    *   B: Ambient Occlusion
3.  **Noise Textures:** Tileable 3D textures [S2, unofficial].
    *   Shape Noise: Fractal Perlin-Worley noise [S2, unofficial].
    *   Detail Noise: Fractal Perlin noise [S2, unofficial].

**Steps:**
1.  **Height Gradient Attenuation:**
    Map the vertical component of $P$ to a normalized height $h \in [0, 1]$ relative to the cloud layer bounds (`baseHeight`, `thickness`) [S1, unofficial]. Apply a vertical remap function to enforce flat cloud bases and rounded tops [S1, unofficial]. This attenuates the base density $\rho_{base}$ [S1, unofficial].
    $$ \rho_{height} = \rho_{base} \times \text{Remap}(h) $$

2.  **Weather Map Sampling:**
    Sample the 2D Cloud Map at the horizontal coordinates of $P$ to retrieve Coverage ($C$), Type ($T$), Erosion ($E_{map}$), and Wetness ($W$) [S1, S2, unofficial].
    *   Map $T$ to specific cloud layers:
        *   $0.0 \le T < 0.125$: Cirrus
        *   $0.125 \le T < 0.25$: Cirrus + Cumulus
        *   $0.25 \le T < 0.5$: Cumulus
        *   $0.5 \le T \le 1.0$: Cumulonimbus [S2, unofficial].

3.  **Base Noise Sampling:**
    Sample the primary 3D Shape Noise texture using low-frequency coordinates scaled to tile across kilometers [S1, unofficial]. This generates the basic cloud shape [S2, unofficial].
    $$ n_{shape} = \text{SampleShapeNoise}(P \times \text{Scale}_{low}) $$

4.  **Erosion and Detail:**
    Sample the secondary Worley noise texture for edge erosion [S1, unofficial] and the secondary Fractal Perlin noise texture for fine detail [S2, unofficial].
    *   Scale the Worley noise frequency up to erode edges [S1, unofficial].
    *   Apply wind advection: offset the noise coordinates by $Wind \times t$ [S1, S2, unofficial].
    *   Calculate erosion factor $E_{total}$ using $E_{map}$ from the Cloud Map and $E_{lut}$ from the Cloud Lut [S2, unofficial]. Note: The Cloud Lut G channel (Erosion) affects both detail and shape noise calculations [S2, unofficial].
    *   For complex shapes like Cumulonimbus, adjust coverage and erosion based on height: high altitude has low coverage and high erosion; low altitude has high coverage and low erosion [S2, unofficial].
    $$ \rho_{eroded} = \rho_{height} \times (1.0 - \text{ErosionFunction}(n_{detail}, n_{worley}, E_{total})) $$

5.  **Final Density Composition:**
    Combine the eroded density with the Coverage map and Cloud Lut density values [S1, unofficial].
    $$ \rho = \rho_{eroded} \times C \times \text{LutDensity}(T) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def SampleCloudDensity(P_world, time, wind, weather_params, cloud_map_tex, cloud_lut_tex, shape_noise_tex, detail_noise_tex):
    # Helper: Linear interpolation [G]
    def lerp(a, b, t):
        return a + t * (b - a)

    # Helper: Smoothstep edge function [G]
    def smoothstep(edge0, edge1, x):
        t = max(0.0, min(1.0, (x - edge0) / (edge1 - edge0)))
        return t * t * (3.0 - 2.0 * t)

    # 1. Height Gradient
    h = (P_world.y - weather_params.baseHeight) / weather_params.thickness
    height_factor = max(0.0, min(1.0, h)) # Vertical remap enforces flat base and rounded tops [S1, unofficial]

    # 2. Weather Map Sampling
    uv = P_world.xz * weather_params.scale_uv
    cloud_map = cloud_map_tex.Sample(uv)
    coverage = cloud_map.r
    cloud_type = cloud_map.g
    erosion_map = cloud_map.b
    wetness = cloud_map.a

    # 3. Base Noise (Shape)
    shape_scale = 1.0 / weather_params.km_scale
    n_shape = shape_noise_tex.Sample(P_world * shape_scale + wind.xz * time)

    # 4. Erosion and Detail
    erosion_scale = shape_scale * weather_params.erosion_frequency_multiplier
    n_worley = detail_noise_tex.Sample(P_world * erosion_scale + wind.xz * time) # Secondary Worley for erosion [S1, unofficial]

    detail_scale = shape_scale * weather_params.detail_frequency_multiplier
    n_detail = detail_noise_tex.Sample(P_world * detail_scale + wind.xz * time) # Fractal Perlin for detail [S2, unofficial]

    lut_erosion = cloud_lut_tex.Sample(cloud_type).g
    total_erosion = lerp(erosion_map, lut_erosion, 0.5) # Combine map and Lut erosion [S2, unofficial]

    if cloud_type > 0.5:
        height_erosion_factor = 1.0 - h
        total_erosion *= (1.0 + height_erosion_factor) # Height-based erosion adjustment for Cumulonimbus [S2, unofficial]

    erosion_mask = smoothstep(0.0, total_erosion, n_worley)
    detail_mask = smoothstep(0.0, total_erosion, n_detail)

    base_density = n_shape * (1.0 - erosion_mask) + n_detail * detail_mask

    # 5. Final Composition
    lut_density = cloud_lut_tex.Sample(cloud_type).r
    final_density = base_density * height_factor * coverage * lut_density

    return final_density
```

## Subtopics

### [Cumulus Cloud Rendering Configuration](2897-cumulus-cloud-rendering-configuration.md)
Defines specific procedural noise parameters for cumulus clouds within the Ray-Guided Streaming framework, tailored for multiple scattering simulations.

### [IsimCloudTexture density evaluation](3530-isimcloudtexture-density-evaluation.md)
Computes density by combining geometric primitives, FBM noise, and depth-based fading via CUDA kernels, outputting grayscale density suitable for volume rendering.

### [Procedural Gas Modeling via Turbulence](3551-procedural-gas-modeling-via-turbulence.md)
Generates gas density using a turbulence function mapped through a precomputed power table to replace expensive power function calls.

### [Cloud Density Sampling](4806-cloud-density-sampling.md)
Generates base volumetric density using weighted sums of low-frequency 3D noise textures (FBM), serving as the foundational density evaluation step.

### [Finite Layer Cloud Shape Generation](8292-finite-layer-cloud-shape-generation.md)
Constructs 3D volume shapes of layer clouds (Cirrus, Stratus, etc.) by assembling connected 3D ellipsoidal lobes to avoid degenerate cylinders.

### [Baked Cloud Density and AO Sampling](16026-baked-cloud-density-and-ao-sampling.md)
Replaces procedural noise with precomputed 3D volume textures containing analytical density and SDFs derived from Worley cells for performance optimization.

### [Physical Cloud Model (Cumulus Convection)](6212-physical-cloud-model-cumulus-convection.md)
A physical model for cumulus convection that applies a constant heat source at the base to simulate sunlight heating the earth. The saturation mixing ratio is an exponential function of altitude: $q_s = A \exp(-\alpha z)$. The liquid water mixing ratio is $q_l = \max(q - q_s, 0)$. Parameters include latent heat of vaporization $L_v$, specific heat of air $c_p$, and friction timescale $t_s$. Boundary conditions set $q_0 = 0.02$ at the bottom and $q_0 = 0.002$ at the top.

### 3D Volume Bake Shader for Volumetric Clouds
A compute shader that computes physical evaluated cloud density at each voxel's Earth-curved world-space coordinate [S3, unofficial]. It uses the full weather map, advection, advection offset, and noise detailing [S3, unofficial]. The evaluated density is stored in both the baseAlbedo (B) and baseDensity (A) channels of the 3D texture [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample, but requires multiple texture lookups (Shape, Detail, Worley, Cloud Map, Cloud Lut) and noise evaluations per ray step [S2, unofficial].
*   **Memory Complexity:** Depends on the resolution of the 3D noise textures and 2D Cloud Map [S2, unofficial]. Noise textures must be tileable [S2, unofficial].
*   **Performance:** Real-time capable on PC/Console hardware [S2, unofficial]. Performance gains are achieved by baking density and AO into 3D textures [S3, unofficial].
*   **Strengths:** Highly flexible; allows real-time control of cloud shape, coverage, and type via parameters [S1, S2, unofficial]. Supports global and local advection [S2, unofficial].
*   **Limitations:** Procedural evaluation is computationally expensive compared to baked textures [S3, unofficial]. Requires careful tuning of noise parameters to avoid artifacts [S2, unofficial].
*   **When to use:** Real-time rendering where dynamic cloud evolution and parameter control are required [S2, unofficial].
*   **When not to use:** When maximum performance is critical and cloud shapes are static (use Baked Cloud Density) [S3, unofficial].

## Background
Originating from techniques used in games like Horizon, Red Dead Redemption 2, and Frostbite, and popularized in Unity HDRP 12.0+ [S2, unofficial]. The method relies on 3D Perlin-Worley noise for shape and Perlin noise for detail [S1, S2, unofficial]. It builds on the principles of volumetric cloud rendering by providing the density field necessary for raymarching and lighting integration [S1, unofficial].

## Sources
[S1] Volumetric Clouds with 3D Perlin-Worley Noise in OpenGL, https://salivity.github.io/opengl/article/volumetric-clouds-with-3-d-perlin-worley-noise-in-open-gl, fetched 2026-10-29 (unofficial)
[S2] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
[S3] Enhance volumetric clouds system with precomputed density, 3D AO and shadow transmittance, https://github.com/ricecake/boidsish/pull/708, fetched 2026-10-27 (unofficial)
