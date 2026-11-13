# Texture Coordinate Conversion
Also called: local2tex

## Overview
Texture Coordinate Conversion is a utility function that maps local or world space coordinates to normalized texture coordinates ($uv$) for volume sampling [S1, unofficial]. It solves the problem of aligning discrete voxel grid indices with continuous texture sampler inputs by scaling by the inverse volume dimensions, applying a 0.5 offset to center samples within texels, and flipping the Y axis to match texture coordinate conventions [S1, unofficial].

## How it works
**Inputs:**
- $P = (P_x, P_y, P_z)$ is the input position vector [S1, unofficial].
- $g\_cf3InvVolSize = (1/N_x, 1/N_y, 1/N_z)$ is a 3D vector containing the inverse dimensions of the volume [S1, unofficial].

**Outputs:**
- $uv = (uv_x, uv_y, uv_z)$ is the resulting texture coordinate vector in the range $[0, 1]$ [S1, unofficial].

**Data Structures:**
- $g\_cf3InvVolSize$ is stored as a `float3` (or equivalent 3-component float vector) [S1, unofficial].

**Steps:**
1. Scale the input position $P$ by the inverse volume size vector $g\_cf3InvVolSize$ [S1, unofficial].
2. Add a constant offset of $0.5$ to each component of the result [S1, unofficial].
3. Flip the Y coordinate by subtracting the current Y value from 1 [S1, unofficial].

**Formula:**
$$uv = P \cdot g\_cf3InvVolSize + 0.5$$
$$uv.y = 1 - uv.y$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def texture_coordinate_conversion(P_x, P_y, P_z, inv_vol_size_x, inv_vol_size_y, inv_vol_size_z):
    """
    Converts local/world space coordinates P to texture coordinates uv.
    """
    uv_x = P_x * inv_vol_size_x + 0.5
    uv_y = P_y * inv_vol_size_y + 0.5
    uv_z = P_z * inv_vol_size_z + 0.5
    uv_y = 1.0 - uv_y
    return uv_x, uv_y, uv_z
```

**Parameters:**
- $g\_cf3InvVolSize$ contains the inverse dimensions of the volume [S1, unofficial]. Typical values depend on the volume resolution; for a $256^3$ volume, this is $(1/256, 1/256, 1/256)$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Boundaries:** If $P$ is outside the volume bounds, $uv$ may exceed $[0, 1]$, requiring clamping or border handling in the sampler [G].
- **Precision:** Floating-point precision errors may occur if the volume dimensions are extremely large or small [G].

## Subtopics

### [Voxel Texture Atlas UV Mapping](4162-voxel-texture-atlas-uv-mapping.md)
Voxel Texture Atlas UV Mapping is a technique that maps voxel face types and directions to a 2D texture atlas using calculated UV coordinates, incorporating safety margins to prevent texture bleeding. It builds on [Texture Coordinate Conversion](3581-texture-coordinate-conversion.md) by extending the coordinate conversion into a structured atlas layout, allowing efficient sampling of multiple voxel types from a single texture resource.

### [Inverse UV Texture](10734-inverse-uv-texture.md)
The Inverse UV Texture is a lookup table used in differentiable rendering to convert gradients from screen space back into texture space, enabling hardware-accelerated backward texture sampling. It solves the difficulty of performing atomic float addition operations required to sum gradients from neighboring pixels, which is not fully supported in Vulkan APIs. Unlike software-based automatic differentiation libraries that use tensor operations or handwritten CUDA modules, this method leverages the graphics pipeline's hardware sampler directly.

### Voxel Center Alignment Offset
Offsets the camera position in texture space by half a unit to align ray origins with voxel centers, assuming texture coordinates range from 0 to 1 [S2, unofficial]. It computes the camera position in object space using the inverse model-view matrix and adds $(0.5, 0.5, 0.5)$ to get the texture space position [S2, unofficial].
**Formula:**
$$\mathbf{p}_{\text{tex}} = \mathbf{p}_{\text{obj}} + (0.5, 0.5, 0.5)$$

### Texture Coordinate Generation for Volume
Generates texture coordinates for the 8 corners of the volume box to map the 3D texture correctly, accounting for half-texel offsets [S3, unofficial]. It computes half-texel offsets ($dx = 0.5/size.x$, etc.) and corner coordinates ($x0 = 0.0 + dx$, $x1 = 1.0 - dx$, etc.) [S3, unofficial]. For sub-regions, it normalizes min/max corners by image size and applies similar offsets [S3, unofficial]. The 8 vertices are assigned in the order: $(x0,y0,z1), (x1,y0,z1), (x1,y1,z1), (x0,y1,z1), (x0,y0,z0), (x1,y0,z0), (x1,y1,z0), (x0,y1,z0)$ [S3, unofficial]. A pitfall noted is uncertainty about why $z1$ is used for the first 4 vertices and $z0$ for the last 4 [S3, unofficial].

### IsoSurface Vertex Shader
The coordinate transformation assumes the unit cube is centered at the origin with min $(-0.5,-0.5,-0.5)$ and max $(0.5,0.5,0.5)$ [S4, unofficial]. It computes 3D texture coordinates by adding $(0.5, 0.5, 0.5)$ to the object space vertex position to map the unit cube to texture space [S4, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(1)$ per coordinate conversion [G].
- Memory: $O(1)$ for the conversion function itself [G].

**Measured Performance:** Not given in the sources.

**Scaling:** Not given in the sources.

**Strengths and Limitations:** Not given in the sources.

**When to use it and when not:** Not given in the sources.

## Background
Originated in the context of dynamic volume raycasting, specifically in the `DynamicVolumeRayCasting` project [S1, unofficial]. The method is implemented in the `RayCast.fx` shader file, which includes headers defining volume dimensions (e.g., $256^3$) and other rendering parameters [S1, unofficial]. It is part of the pipeline that prepares ray positions for sampling from a `Texture3D` resource [S1, unofficial].

## Sources
[S1] DynamicVolumeRayCasting, https://github.com/pengliu916/DynamicVolumeRayCasting, fetched 2026-10-26 (unofficial)
[S2] qt-viewer-pro, https://github.com/wpop/qt-viewer-pro, fetched 2026-10-27 (unofficial)
[S3] RaycastVolume.cpp, https://visualizationlibrary.org/documentation/_raycast_volume_8cpp_source.html, fetched 2026-10-27 (unofficial)
[S4] Volumetric Render Engine - GitHub, https://github.com/mikejernil/volumetric-render-engine, fetched 2026-10-27 (unofficial)
