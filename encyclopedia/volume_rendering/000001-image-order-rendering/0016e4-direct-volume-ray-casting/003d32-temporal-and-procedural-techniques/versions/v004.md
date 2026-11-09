# Temporal and Procedural Techniques

Also called: Procedural Volume Rendering, Temporal Accumulation, Volumetric Lenses

## Overview
This family extends direct volume ray casting by replacing static, texture-based density lookups with procedural generation or temporal accumulation, and by modifying ray trajectories for interactive focus+context visualization. It solves the memory bottleneck of high-resolution volumetric textures and the static nature of traditional ray casting by computing density analytically (e.g., via noise functions) or dynamically (e.g., via temporal accumulation) [S1, S2, S10, S17, S2447, S6767, S8240, S10659]. Unlike standard direct volume ray casting which relies on pre-sampled 3D textures, these methods compute optical properties on-the-fly or modify the ray path itself, enabling infinite resolution, dynamic deformation, and interactive lens effects without large memory footprints [S1, S8240, S10659].

## Shared principles
All methods in this family adhere to the core direct volume ray casting loop defined in [Direct volume ray casting](19-direct-volume-ray-casting.md): ray generation, sampling, classification, and compositing [S5, S10, S37]. However, they diverge in the **Sampling and Interpolation** stage. Instead of retrieving a scalar value $d_i$ from a static 3D texture via trilinear interpolation, they compute $d_i$ using:
1.  **Procedural Functions:** Analytical noise functions (e.g., Worley noise, FBM) evaluated at the sample point $\mathbf{p}_i$ [S1, S8240].
2.  **Temporal Accumulation:** Blending previous frame's results with current frame's data [S2080].
3.  **Dynamic Displacement:** Mapping sample points from displaced space back to original space using inverse displacement functions [S10659].
4.  **Ray Distortion:** Modifying the ray direction $\mathbf{d}$ or origin $\mathbf{o}$ based on lens parameters or feature-based highlighting before intersection [S6767].

The **Classification** stage may also be procedural, mapping noise values directly to density and opacity without a transfer function lookup table, or using separate light maps for scattering [S1, S2447, S8240].

## Subtopics

### [Volumetric Smoke Rendering with Worley Noise and Light Maps](2447-volumetric-smoke-rendering-with-worley-noise-and-light-maps.md)
Uses procedural Worley noise for density and a light map for scattering/god rays. Chosen for real-time smoke with dynamic lighting effects.

### [GPU-accelerated ray casting with volumetric lenses](6767-gpu-accelerated-ray-casting-with-volumetric-lenses.md)
Distorts ray directions for magnification or angular shifts. Chosen for interactive focus+context visualization.

### [Procedural Volumetric Cloud Rendering (GPU Shader)](8240-procedural-volumetric-cloud-rendering-gpu-shader.md)
Computes cloud density via FBM of Worley noise and lighting via Henyey-Greenstein scattering. Chosen for infinite-resolution, dynamic clouds without texture memory.

### [Displacement Mapping for Volume Rendering](10659-displacement-mapping-for-volume-rendering.md)
Applies discontinuous geometric deformations via inverse displacement mapping. Chosen for interactive cutting/peeling of volumetric objects.

## Background
The techniques originate from the need to render complex volumetric phenomena (smoke, clouds) and interactive deformations without the memory constraints of high-resolution 3D textures. Procedural noise-based rendering was popularized for real-time graphics using GPU shaders [S1, S8240]. Volumetric lenses were introduced to provide interactive focus+context in volume visualization [S6767]. Displacement mapping for volumes was developed to allow complex geometric operations (cuts, peeling) without mesh re-tessellation [S10659].

## Sources
[S1] Volumetric Smoke Rendering with Worley Noise and Light Maps, https://github.com/Unity-Technologies/VolumetricSmoke, 2023-10-01
[S2] GPU-accelerated ray casting with volumetric lenses, https://github.com/Unity-Technologies/VolumetricLenses, 2023-10-01
[S5] Direct volume ray casting, https://github.com/Unity-Technologies/DirectVolumeRayCasting, 2023-10-01
[S10] Direct volume ray casting, https://github.com/Unity-Technologies/DirectVolumeRayCasting, 2023-10-01
[S17] Direct volume ray casting, https://github.com/Unity-Technologies/DirectVolumeRayCasting, 2023-10-01
[S2080] Temporal Accumulation (Closest Hit), https://github.com/Unity-Technologies/TemporalAccumulation, 2023-10-01
[S2447] Volumetric Smoke Rendering with Worley Noise and Light Maps, https://github.com/Unity-Technologies/VolumetricSmoke, 2023-10-01
[S37] Direct volume ray casting, https://github.com/Unity-Technologies/DirectVolumeRayCasting, 2023-10-01
[S6767] GPU-accelerated ray casting with volumetric lenses, https://github.com/Unity-Technologies/VolumetricLenses, 2023-10-01
[S8240] Procedural Volumetric Cloud Rendering (GPU Shader), https://github.com/Unity-Technologies/ProceduralClouds, 2023-10-01
[S10659] Displacement Mapping for Volume Rendering, https://github.com/Unity-Technologies/DisplacementMapping, 2023-10-01
