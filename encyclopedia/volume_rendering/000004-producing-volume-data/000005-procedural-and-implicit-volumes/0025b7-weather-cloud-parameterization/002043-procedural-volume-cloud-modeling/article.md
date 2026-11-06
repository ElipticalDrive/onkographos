# Procedural Volume Cloud Modeling

Also called: Cloud Density Field Generation, Density Sampling and Weather

## Overview
Procedural Volume Cloud Modeling constructs a volumetric density field for clouds by evaluating analytic functions and sampling noise textures at arbitrary world-space coordinates, rather than relying on pre-baked voxel grids or physical simulations [S1, S2, unofficial]. It solves the problem of generating realistic, large-scale cloud formations with varying types (e.g., cumulus, cirrus) and dynamic details (erosion, flow) efficiently on GPU hardware [S2, unofficial]. It differs from [Volumetric Cloud Raymarching](1248-volumetric-cloud-raymarching.md) by focusing on the *generation* of the density values themselves rather than the integration of light through them, and from [Atmospheric Height Gradient](4813-atmospheric-height-gradient.md) by incorporating multi-channel weather maps and specific cloud-type logic rather than a simple vertical falloff [S1, S2, unofficial].

## How it works
The method maps a 3D world-space position to a scalar density value (and optionally color/occlusion) using a combination of height-based attenuation, weather map lookups, and fractal noise sampling.

### Inputs and Outputs
*   **Inputs**:
    *   `P`: World-space position $(x, y, z)$ [S1, S2, unofficial].
    *   `t`: Simulation time [S1, S2, unofficial].
    *   `Wind`: 3D wind vector $(u, v, w)$ for advection [S1, S2, unofficial].
    *   `CloudMap`: 2D texture storing coverage, type, erosion, and wetness [S2, unofficial].
    *   `ShapeNoise`: Tileable 3D texture containing fractal Perlin-Worley noise [S2, unofficial].
    *   `DetailNoise`: Tileable 3D texture containing fractal Perlin noise [S2, unofficial].
    *   `CloudLut`: 2D texture mapping cloud type/height to density, erosion, and ambient occlusion [S2, unofficial].
*   **Outputs**:
    *   `Density`: Scalar cloud density value [S1, S2, unofficial].
    *   `Color`: Optional RGB color derived from density and occlusion [S2, unofficial].

### Data Structures
1.  **CloudMap** (2D Texture, RGBA):
    *   `R`: Cloud Coverage (0.0–1.0) [S2, unofficial].
    *   `G`: Cloud Type (0.0–1.0). Mapped to layers: 0–0.125 Cirrus, 0.125–0.25 Cirrus+Cumulus, 0.25–0.5 Cumulus, 0.5–1.0 Cumulonimbus [S2, unofficial].
    *   `B`: Cloud Erosion (0.0–1.0) [S2, unofficial].
    *   `A`: Cloud Wetness (0.0–1.0) [S2, unofficial].
2.  **CloudLut** (2D Texture, RGB):
    *   `R`: Density [S2, unofficial].
    *   `G`: Erosion [S2, unofficial].
    *   `B`: Ambient Occlusion [S2, unofficial].
3.  **Noise Textures**:
    *   Must be tileable to support large-scale tiling across kilometers [S2, unofficial].

### Algorithm Steps
1.  **Sample Weather Map**:
    *   Project `P` to 2D (typically XZ plane) to get UV coordinates `UV`.
    *   Sample `CloudMap` at `UV` to get `Coverage`, `CloudType`, `ErosionMap`, and `Wetness` [S1, S2, unofficial].
2.  **Height Gradient Attenuation**:
    *   Apply a vertical remap function to `P.y` to enforce flat cloud bases and rounded tops. This attenuates density based on altitude [S1, unofficial].
    *   *Note*: For complex types like Cumulonimbus, adjust noise coverage and erosion based on height: high altitude has low coverage and high erosion; low altitude has high coverage and low erosion [S2, unofficial].
3.  **Base Noise Sampling (Shape)**:
    *   Calculate shape noise coordinates `UV_shape`.
    *   Apply wind advection: `UV_shape += Wind.xz * t * Scale` [S1, S2, unofficial].
    *   Sample `ShapeNoise` (Fractal Perlin-Worley) at `UV_shape` to get `BaseShape` [S1, S2, unofficial].
    *   Scale `BaseShape` by `Coverage` from the Weather Map [S2, unofficial].
4.  **Detail Noise Sampling (Erosion)**:
    *   Calculate detail noise coordinates `UV_detail`.
    *   Apply wind advection: `UV_detail += Wind.xz * t * DetailScale` [S1, S2, unofficial].
    *   Sample `DetailNoise` (Fractal Perlin) at `UV_detail` to get `Detail` [S1, S2, unofficial].
    *   Combine `Detail` with `ErosionMap` from Weather Map and `Erosion` from `CloudLut` (G channel) to erode the edges of `BaseShape` [S1, S2, unofficial].
    *   *Pitfall*: The `CloudLut` G channel (Erosion) affects both detail noise and shape noise calculations [S2, unofficial].
5.  **Final Density Calculation**:
    *   Combine the eroded shape noise with height attenuation and `CloudLut` density (R channel) based on `CloudType` [S1, S2, unofficial].
    *   Result is the final `Density` [S1, S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def SampleCloudDensity(P: Vec3, t: float, Wind: Vec3, 
                       CloudMap: Texture2D, ShapeNoise: Texture3D, 
                       DetailNoise: Texture3D, CloudLut: Texture2D) -> float:
    # 1. Sample Weather Map (Project P to XZ plane)
    UV = Vec2(P.x, P.z)
    weather = CloudMap.Sample(UV)
    coverage = weather.R
    cloud_type = weather.G
    erosion_map = weather.B
    
    # 2. Height Gradient
    # Enforce flat base and rounded top. 
    # Specific logic for Cumulonimbus (type > 0.5): 
    # High altitude -> low coverage, high erosion.
    # Low altitude -> high coverage, low erosion.
    height_factor = ApplyHeightGradient(P.y, cloud_type)
    
    # 3. Base Noise (Shape)
    # Low frequency, tiling across kilometers
    shape_scale = 0.001 # Example scale
    shape_uv = P.xz * shape_scale
    shape_uv += Wind.xz * t * 0.1 # Global wind advection
    
    # Fractal Perlin-Worley noise
    base_shape = SampleFractalPerlinWorley(ShapeNoise, shape_uv)
    
    # Apply coverage and cloud type influence
    # CloudLut R channel provides base density for the type
    lut_density = CloudLut.Sample(Vec2(cloud_type, 0.0)).R
    shape_density = base_shape * coverage * lut_density
    
    # 4. Detail Noise (Erosion)
    # Higher frequency, erodes edges
    detail_scale = 0.01 # Example scale
    detail_uv = P.xz * detail_scale
    detail_uv += Wind.xz * t * 0.2 # Faster wind for details
    
    # Fractal Perlin noise
    detail = SampleFractalPerlin(DetailNoise, detail_uv)
    
    # Combine erosion sources
    # CloudLut G channel affects both shape and detail [S2, unofficial]
    lut_erosion = CloudLut.Sample(Vec2(cloud_type, 0.0)).G
    total_erosion = max(erosion_map, lut_erosion)
    
    # Erode the shape
    eroded_shape = shape_density * (1.0 - detail * total_erosion)
    
    # 5. Final Density
    final_density = eroded_shape * height_factor
    
    return max(0.0, final_density)

def ApplyHeightGradient(y: float, cloud_type: float) -> float:
    # Simplified height remap
    # Flat base at y=0, rounded top
    if y < 0.0:
        return 0.0
    # Example Gaussian-like falloff for top
    return exp(-pow(y, 2))
```

### Parameters
*   **Noise Scale**: Low frequency for shape (kilometer scale), high frequency for detail [S1, S2, unofficial].
*   **Wind Speed**: Controls advection rate for shape and detail noise [S1, S2, unofficial].
*   **Tileable**: Noise textures must be tileable to avoid seams at boundaries [S2, unofficial].

### Edge Cases and Pitfalls
*   **Seams**: Non-tileable noise textures cause visible seams in large clouds [S2, unofficial].
*   **Erosion Interaction**: The `CloudLut` G channel (Erosion) influences both detail and shape noise; incorrect mapping leads to unnatural cloud structures [S2, unofficial].
*   **Height Logic**: Cumulonimbus clouds require height-dependent coverage and erosion adjustments to avoid uniform cylindrical shapes [S2, unofficial].

## Subtopics

### [Cumulus Cloud Rendering Configuration](2897-cumulus-cloud-rendering-configuration.md)
Defines specific procedural noise parameters for cumulus clouds within the Ray-Guided Streaming framework, tailored for multiple scattering simulations.

### [IsimCloudTexture density evaluation](3530-isimcloudtexture-density-evaluation.md)
Computes density using geometric primitives, FBM noise, and depth-based fading via CUDA kernels, outputting grayscale density suitable for volume rendering.

### [Procedural Gas Modeling via Turbulence](3551-procedural-gas-modeling-via-turbulence.md)
Generates gas density using a turbulence function and a precomputed power table lookup to replace expensive power function calls.

### [Cloud Density Sampling](4806-cloud-density-sampling.md)
Generates base density using weighted sum of low-frequency 3D noise textures (FBM) and remapping, handling wind skew and basic height gradients.

### [Finite Layer Cloud Shape Generation](8292-finite-layer-cloud-shape-generation.md)
Constructs 3D volume shapes of layer clouds (Cirrus, Stratus, etc.) by assembling connected 3D ellipsoidal lobes to ensure vertical presence variation and horizontal tapering.
*   Physical Cloud Model (Cumulus Convection): A physical simulation approach applying a constant heat source at the base, calculating evaporation and condensation based on saturation mixing ratio $q_s = A \exp(-\alpha z)$ and liquid water mixing ratio $q_l = \max(q - q_s, 0)$, ignoring Coriolis effects.

## Operating characteristics
*   **Time Complexity**: $O(1)$ per sample, as it involves texture lookups and mathematical operations. No simulation steps are required [S1, S2, unofficial].
*   **Memory Complexity**: Depends on the resolution of the noise textures and CloudMap/CloudLut. Typically fits in GPU VRAM [S2, unofficial].
*   **Performance**: Highly efficient for real-time rendering on PC/Console hardware, as it avoids expensive voxelization or physical simulation [S2, unofficial].
*   **Strengths**: Flexible, supports dynamic wind and time-based animation, allows artistic control via textures [S1, S2, unofficial].
*   **Limitations**: Lacks physical consistency (e.g., mass conservation) compared to simulation-based methods [S3, unofficial]. Requires careful tuning of noise parameters to avoid repetitive patterns [S2, unofficial].

## Background
The method originated from real-time rendering requirements for PC and console games, with notable implementations in Unity HDRP (version 12.0+) and games like Horizon and Red Dead Redemption 2 [S2, unofficial]. It evolved from earlier static cloud textures to dynamic, noise-based procedural generation to support large-scale, animated cloud layers [S2, unofficial]. The physical cloud model variant traces back to SIGGRAPH '84 work on ray tracing volume densities, which used a convective model with latent heat of vaporization and frictional effects [S3, unofficial].

## Sources
[S1] Volumetric Clouds with 3D Perlin-Worley Noise in OpenGL, https://salivity.github.io/opengl/article/volumetric-clouds-with-3-d-perlin-worley-noise-in-open-gl, fetched 2026-10-29 (unofficial)
[S2] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
[S3] Ray Tracing for Volume Densities, https://www.scribd.com/document/746685923/ray-tracking-volume-densities, fetched 2026-10-25 (unofficial)
