# 3D Texture Generation

Also called: 3D Texture Family, Volumetric Texture Mapping

## Overview
This family encompasses methods for storing volumetric scalar field data directly into a 3D texture object, allowing the graphics hardware to perform volume rendering operations such as ray casting or slicing via hardware-accelerated texture lookups [S1][S2]. It solves the problem of high computational cost associated with per-voxel integration on the CPU or in custom shaders by offloading the volume composition operator and interpolation to the GPU's texture unit [S2]. It differs from sibling methods like [View-Aligned Slicing with 3D Textures](870-view-aligned-slicing-with-3d-textures.md) by treating the volume as a continuous 3D resource accessed via 3D texture coordinates, rather than generating explicit 2D slice geometry for rasterization [S1][S2].

## Shared principles
The 3D texture generation family shares the following mechanism, terms, and mathematical foundations:

1.  **Volumetric Data Storage:** The scalar field is stored in a 3D texture object with dimensions $W \times H \times D$. Each texel stores RGBA values, where RGB represents color and A represents opacity [S1].
2.  **Hardware Interpolation:** The graphics hardware performs trilinear interpolation on the 3D texture to sample color and opacity values along the ray path or slice plane [S1].
3.  **Linear Mapping:** The method relies on linear mapping to place color and opacity in the 3D texture, avoiding non-linear formulae typically associated with depth integration [S1].
4.  **Hardware Compositing:** The fragments are composited into the frame buffer using hardware-supported alpha blending. The standard blending equation is:
    $$ C_{out} = C_{in} + C_{sample} \cdot \alpha_{sample} \cdot (1 - \alpha_{in}) $$
    $$ \alpha_{out} = \alpha_{in} + \alpha_{sample} \cdot (1 - \alpha_{in}) $$
    where $C$ is color and $\alpha$ is opacity [S1]. This implements the discrete volume rendering integral when slices are processed in correct depth order [S3, book].

## Subtopics

### [AR mapping to 3D texture](884-ar-mapping-to-3d-texture.md)
A technique that stores scalar field data directly into a 3D texture, allowing hardware ray casting/slicing. It differs by avoiding explicit 2D slice image generation and treating the volume as a continuous 3D resource.

### [3D Texture Generation via Slicing](4891-3d-texture-generation-via-slicing.md)
A method for constructing a 3D texture by rendering 2D slices to a 2D texture, reading pixels, and assembling them into a 3D texture. It is one of the simplest methods for volume visualization.

### [Post-packing Optimisation](11679-post-packing-optimisation.md)
An optimization step that standardizes slice orientations to reduce memory padding waste caused by mixed aspect ratios in the final 3D texture.

### Texture Import and Slicing (Unity)
- **Step:** Import `noiseShape.tga` and `noiseErosion.tga` into Unity. Set Format to RGBA 32 Bit, disable alpha is transparency, disable generate mipmaps, disable sRGB, enable read-write, and set max size to 4096 (or higher via script) [S1]. Split the 2D textures into 3D textures using the `SplitSelectedTexture` function (`Tools/Textures/Split Texture2D To Texture3D Cube`). This creates `NoiseShape.asset` and `NoiseErosion.asset` Texture3D assets [S1].
- **Pitfall:** `noiseShape.tga` has a width of 16384, which exceeds the default Unity max size. Use `Tools/Textures/Set Max Size To 16384` to allow full-size import [S1].

### Context Volume Rendering
- **Definition:** Rendering large volumes by maintaining a full-resolution 3D texture for MPR (Multi-Planar Reformation) views and a reduced-resolution 'context volume' for the 3D pane to save GPU memory [S2].
- **Step:** Calculate context volume dimensions as half per axis (1/2 reduction) relative to the full-resolution volume [S2].
- **Parameter:** `maxTextureDimension3D` runtime limit. Default WebGPU guarantee is 2048, but actual adapter limits may differ. Must compare each volume dimension against `device.limits.maxTextureDimension3D` [S2].
- **Pitfall:** Context volume still requires significant GPU memory (~1 GB for 1024×1024×4096). End-to-end renderability depends on runtime GPU memory, CPU cache capacity, and active MPR views. The column 'Renderable with context?' should be renamed to '3D texture payload fits' and made conditional [S2]. If even the context volume exceeds limits, a second reduction level or bricking with a page table is required for out-of-core data [S2].
- **Number:** Memory usage for 512×512×400: 210 MB full-res, 26 MB context. For 512×512×2000: 1.05 GB full-res, 131 MB context. For 1024×1024×2000: 4.2 GB full-res, 524 MB context. For 1024×1024×4096: 8.6 GB full-res, 1.05 GB context [S2].

## Background
The concept of using 3D textures for volume rendering was introduced in the early 1990s as a way to leverage emerging 3D graphics hardware. The specific implementation of "AR mapping to 3D texture" is described in early literature on volume visualization using texture mapping [S1]. The technique of generating 3D textures via slicing is described as one of the simplest methods for volume visualization [S1]. Modern implementations, such as those in Unity and WebGPU-based renderers, continue to use these principles for efficient volume rendering [S1][S2].

## Sources
[S1] Noise Textures, https://github.com/TylerDodds/VolumetricCloudsTutorial/blob/master/Assets/VolumetricCloudsTutorial/CloudDensity/NoiseTextures/NoiseTextures.md, fetched 2026-10-29
[S2] docs(volume3d): analysis of the three Volume3D render paths and an improvement plan, https://github.com/cornerstonejs/cornerstone3D/pull/2853, fetched 2026-10-26
