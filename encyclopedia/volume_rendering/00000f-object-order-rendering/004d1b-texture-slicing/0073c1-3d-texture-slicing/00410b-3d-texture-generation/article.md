# 3D Texture Generation
## Overview
This family encompasses methods for storing volumetric scalar field data directly into a 3D texture object, allowing the graphics hardware to perform volume rendering operations such as ray casting or slicing via hardware-accelerated texture lookups [S1, S2]. It solves the problem of high computational cost associated with per-voxel integration on the CPU or in custom shaders by offloading the volume composition operator and interpolation to the GPU's texture unit [S2]. Unlike sibling methods that may use post-packing, isotropic synthesis, or specific engine integrations, this family focuses on the fundamental mechanism of populating a 3D texture array for direct hardware sampling [S1, S2].

## Shared principles
(Empty)

## Subtopics

### [AR mapping to 3D texture](884-ar-mapping-to-3d-texture.md)
Stores scalar field data as color and opacity values within a 3D texture object to facilitate hardware-accelerated rendering. It is distinguished by its reliance on pre-defined linear texture coordinates for parallel polygons, which allows the geometry to be cached in a display list. Choose it when caching geometry and avoiding non-linear depth integration formulae is prioritized.

### [3D Texture Generation via Slicing](4891-3d-texture-generation-via-slicing.md)
Constructs a 3D texture object from volumetric data by rendering 2D slices of the scene to a 2D texture, reading the resulting pixels, and assembling them into a 3D texture. It is described as one of the simplest methods for volume visualization. Choose it for straightforward voxel data generation without post-packing optimizations.

### [Post-packing Optimisation](11679-post-packing-optimisation.md)
A corrective stage applied after greedy packing to optimize inefficiently packed layouts by enforcing a global slice orientation. It addresses memory waste caused by mixing tall and wide slices in the final 3D texture. Choose it when initial greedy packing fails or increases memory requirements.

### [Isotropic Texture Synthesis](13939-isotropic-texture-synthesis.md)
Generates volumetric textures that exhibit uniform distribution in 3D space, with no dependency on position or orientation. It sets itself apart by using standard 2D texture synthesis on a single cross-section, avoiding the complexity and performance costs of full 3D volumetric synthesis. Choose it for materials like sponges or sausages where cross-sections appear similar regardless of location or cutting angle.

### Texture Import and Slicing (Unity)
- **Step:** Import `noiseShape.tga` and `noiseErosion.tga` into Unity. Set Format to RGBA 32 Bit, disable alpha is transparency, disable generate mipmaps, disable sRGB, enable read-write, and set max size to 4096 (or higher via script) [S1]. Split the 2D textures into 3D textures using the `SplitSelectedTexture` function (`Tools/Textures/Split Texture2D To Texture3D Cube`) [S1]. This creates `NoiseShape.asset` and `NoiseErosion.asset` Texture3D assets [S1].
- **Pitfall:** `noiseShape.tga` has a width of 16384, which exceeds the default Unity max size. Use `Tools/Textures/Set Max Size To 16384` to allow full-size import [S1].

### Context Volume Rendering
- **Definition:** Rendering large volumes by maintaining a full-resolution 3D texture for MPR (Multi-Planar Reformation) views and a reduced-resolution 'context volume' for the 3D pane to save GPU memory [S2].
- **Step:** Calculate context volume dimensions as half per axis (1/2 reduction) relative to the full-resolution volume [S2].
- **Parameter:** `maxTextureDimension3D` runtime limit. Default WebGPU guarantee is 2048, but actual adapter limits may differ [S2]. Must compare each volume dimension against `device.limits.maxTextureDimension3D` [S2].
- **Pitfall:** Context volume still requires significant GPU memory (~1 GB for 1024×1024×4096). End-to-end renderability depends on runtime GPU memory, CPU cache capacity, and active MPR views [S2]. The column 'Renderable with context?' should be renamed to '3D texture payload fits' and made conditional [S2]. If even the context volume exceeds limits, a second reduction level or bricking with a page table is required for out-of-core data [S2].
- **Number:** Memory usage for 512×512×400: 210 MB full-res, 26 MB context. For 512×512×2000: 1.05 GB full-res, 131 MB context [S2]. For 1024×1024×2000: 4.2 GB full-res, 524 MB context [S2]. For 1024×1024×4096: 8.6 GB full-res, 1.05 GB context [S2].

## Background
Not given in the sources.

## Sources
[S1] Noise Textures, https://github.com/TylerDodds/VolumetricCloudsTutorial/blob/master/Assets/VolumetricCloudsTutorial/CloudDensity/NoiseTextures/NoiseTextures.md, fetched 2026-10-29
[S2] docs(volume3d): analysis of the three Volume3D render paths and an improvement plan, https://github.com/cornerstonejs/cornerstone3D/pull/2853, fetched 2026-10-26
