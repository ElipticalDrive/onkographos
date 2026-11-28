# Froxel Volume (Frustum Grid)

Also called: Froxel Volumetric Rendering, Frustum Grid, Frustum Volumes, Volumetric Fog Grid

## Overview
Froxel Volume is a real-time volumetric rendering technique that stores volumetric data in a view-frustum-aligned 3D texture grid (froxels), replacing per-pixel raymarching to reduce computational cost [S1, S2, unofficial]. It optimizes for view-dependent rendering by using smaller voxels near the camera and larger ones at a distance, covering both large and small scale effects with a single data model [S1, S2, unofficial]. Unlike per-pixel raymarching, which scales with directional lights and pixel count, froxel rendering costs a single pass over the grid regardless of resolution [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Camera frustum parameters (near and far planes), volumetric density and phase function data [S2, unofficial].
- **Output:** A 3D texture (froxel buffer) containing precomputed volumetric properties (density, scattering, absorption) and a final composited image derived from a single lookup per pixel [S1, S2, unofficial].

**Data Structures:**
- **Froxel Buffer:** An immutable 3D texture allocated via `glTexStorage3D` with target `GL_TEXTURE_3D` [S2, unofficial].
- **Dimensions:** $D_x \times D_y \times D_z$.
    - $D_x, D_y$: Map linearly to normalized device coordinates (screen space). Typical values: 160×90 or 256×144 [S2, unofficial].
    - $D_z$: Distributed exponentially between near and far planes. Typical value: 64 or 128 [S2, unofficial].
    - Default project resolution: 64×64×64 [S1, unofficial].
- **Coordinate Mapping:** View-space depth $z$ maps to normalized texture coordinate $w$ via:
  $$w = \frac{\ln(z / z_{near})}{\ln(z_{far} / z_{near})}$$
  where $z_{near}$ and $z_{far}$ are the camera's near and far clipping plane distances [S2, unofficial].

**Steps:**
1. **Allocate Froxel Buffer:** Create a 3D texture with dimensions $D_x \times D_y \times D_z$ using `glTexStorage3D` [S2, unofficial].
2. **Populate Froxels (Compute Pass):** Dispatch a compute shader (`glDispatchCompute`) to evaluate fog density and material properties for every cell in the 3D texture [S2, unofficial].
    - For uniform fog, density is constant.
    - For dynamic/patchy fog, sample an auxiliary 3D noise texture (e.g., Perlin-Worley) using the reconstructed world-space position of the froxel [S2, unofficial].
    - Compute scattering ($\sigma_s$) and absorption ($\sigma_a$) coefficients based on medium parameters [S2, unofficial].
3. **Render Scene:** Perform a single lookup into the froxel buffer for mesh lookup, rather than marching through world space again [S1, unofficial].
4. **Temporal Reprojection:** Apply temporal reprojection against the last frame to stabilize the result with minimal performance cost [S1, unofficial].

**Parameters:**
- **Resolution:** $D_x, D_y$ typically 160×90 or 256×144; $D_z$ typically 64 or 128 [S2, unofficial]. Default: 64×64×64 [S1, unofficial].
- **Depth Distribution:** Exponential (logarithmic) in Z [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Geometry Shaders:** Avoid using geometry shaders to generate froxels, as they are poorly supported on many hardware platforms, especially integrated graphics [S1, unofficial].
- **Distance Limitations:** Froxel grids typically cover a limited distance (e.g., <160 m). Beyond this, the far slice thickness becomes too large, causing instability even with temporal integration. A secondary raymarcher using the same data model may be needed for distant volumes [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(D_x \cdot D_y \cdot D_z)$ for the single grid pass, independent of screen resolution and directional light count [S1, unofficial].
- **Memory Complexity:** Not given in the sources.
- **Performance:** Significantly faster than per-pixel raymarching for high-resolution screens and multiple lights [S1, unofficial]. Temporal reprojection adds minimal overhead [S1, unofficial].
- **Strengths:** Unified data model for large and small scale effects; view-dependent optimization; stable with temporal integration [S1, S2, unofficial].
- **Limitations:** Limited effective distance due to slice thickness; requires fallback raymarcher beyond ~160 m [S1, unofficial].
- **When to Use:** Real-time volumetric fog/clouds where performance is critical and view-dependent detail is needed [S1, S2, unofficial].
- **When Not to Use:** Scenes requiring extremely high fidelity at large distances without fallback mechanisms [S1, unofficial].

## Background
Froxel-based rendering was notably implemented in Red Dead Redemption 2 as part of a unified volumetric system, aiming to cover large and small scale effects with a single data model [S1, unofficial]. The technique is also used in engines like Godot 4 for volumetric fog, where it replaces per-pixel raymarching to improve performance on integrated graphics [S1, unofficial]. The concept of frustum-aligned grids for volumetrics has been discussed in OpenGL tutorials and technical articles [S2, unofficial].

## Sources
[S1] feat(render): volumetric clouds as a spherical shell, per planet, https://github.com/lobinuxsoft/kooch/issues/731, fetched 2026-10-26 (unofficial)
[S2] Volumetric Fog Using 3D Textures in OpenGL - opengl, https://salivity.github.io/opengl/article/volumetric-fog-using-3-d-textures-in-open-gl, fetched 2026-10-27 (unofficial)
