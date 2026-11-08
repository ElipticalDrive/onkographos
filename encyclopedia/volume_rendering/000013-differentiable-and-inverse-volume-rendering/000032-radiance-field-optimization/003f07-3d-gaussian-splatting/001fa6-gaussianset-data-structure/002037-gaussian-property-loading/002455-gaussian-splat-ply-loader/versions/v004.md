# Gaussian Splat PLY Loader

Also called: PLYLoader::load

## Overview
The Gaussian Splat PLY Loader parses 3D Gaussian Splatting scene data from a Polygon File Format (PLY) file, decoding raw vertex properties into a structured Gaussian representation suitable for rendering. It solves the problem of converting the standardized, often compressed or logarithmically encoded PLY storage format into the explicit mathematical parameters (position, scale, rotation, color, opacity) required by the rasterization pipeline. It differs from its siblings by focusing on the initial ingestion and decoding of the PLY file into the `GaussianData` structure, whereas [Coordinate System Conversion for Gaussian Splats](3935-coordinate-system-conversion-for-gaussian-splats.md) handles spatial alignment and [INRIA V1 PLY Splat Parsing](6822-inria-v1-ply-splat-parsing.md) addresses specific legacy format variations.

## How it works
**Inputs and Outputs:**
- Input: A PLY file containing vertex properties (`x`, `y`, `z`, `scale_0`, `scale_1`, `scale_2`, `rot_0`, `rot_1`, `rot_2`, `rot_3`, `f_dc_0`, `f_dc_1`, `f_dc_2`, `red`, `green`, `blue`, `opacity`).
- Output: A populated `GaussianData` structure containing arrays for positions, scales, rotations, and colors, followed by a packed GPU-ready buffer.

**Data Structures:**
The loader populates the following arrays, each of size `vertexCount`:
- `positions`: `glm::vec3` (3 floats).
- `scales`: `glm::vec3` (3 floats, decoded from log-space).
- `rotations`: `glm::quat` (4 floats, normalized).
- `colors`: `glm::u8vec4` (4 unsigned chars: R, G, B, A).

**Step-by-Step Method:**

1.  **Property Detection:**
    - Request vertex properties `red`, `green`, `blue`. If successful, set `hasRGB = true`. If not, catch exception and leave `hasRGB = false` [S1, unofficial].
    - Request vertex property `opacity`. If successful, set `opacity_ptr` to the data pointer. If not, leave `opacity_ptr` as `nullptr` [S1, unofficial].

2.  **Data Loading:**
    - Read the PLY file data into a `stringstream` or buffer.
    - Determine `vertexCount` from the count of `vertices_x` [S1, unofficial].
    - Resize internal arrays (`positions`, `scales`, `rotations`, `colors`) to `vertexCount` [S1, unofficial].

3.  **Buffer Interpretation:**
    - Reinterpret buffer pointers for coordinates `x`, `y`, `z` as `const float*` (`x_data`, `y_data`, `z_data`) [S1, unofficial].
    - Reinterpret buffer pointers for scale components `scale_0`, `scale_1`, `scale_2` as `const float*` (`scale_0_data`, etc.) [S1, unofficial].
    - Reinterpret buffer pointers for rotation quaternion components `rot_0`, `rot_1`, `rot_2`, `rot_3` as `const float*` (`rot_0_data`, etc.) [S1, unofficial].
    - If spherical harmonics are present (`hasSH`), reinterpret `f_dc_0`, `f_dc_1`, `f_dc_2` as `const float*` (`f_dc_0_data`, etc.) [S1, unofficial].
    - If `hasRGB` is true, reinterpret `red`, `green`, `blue` as `const int*` (`red_data`, etc.) [S1, unofficial].
    - If `opacity_ptr` is not null, it points to `const float*` (`opacity_data`) [S1, unofficial].

4.  **Decoding Loop (for each vertex `i` from 0 to `vertexCount - 1`):**
    - **Position:** Set `positions[i] = glm::vec3(x_data[i], y_data[i], z_data[i])` [S1, unofficial].
    - **Scale:** Decode logarithmic scales. Set `scales[i] = glm::vec3(std::exp(scale_0_data[i]), std::exp(scale_1_data[i]), std::exp(scale_2_data[i]))` [S1, unofficial].
    - **Rotation:** Construct quaternion `q` from `(rot_0_data[i], rot_1_data[i], rot_2_data[i], rot_3_data[i])`. Normalize `q` and store in `rotations[i]` [S1, unofficial].
    - **Color:**
        - If `hasSH` is true: Compute RGB using DC spherical harmonics coefficients.
          $$ r = \text{clamp}((0.5f + SH\_C0 \times f\_dc\_0\_data[i]) \times 255.0f, 0.0f, 255.0f) $$
          $$ g = \text{clamp}((0.5f + SH\_C0 \times f\_dc\_1\_data[i]) \times 255.0f, 0.0f, 255.0f) $$
          $$ b = \text{clamp}((0.5f + SH\_C0 \times f\_dc\_2\_data[i]) \times 255.0f, 0.0f, 255.0f) $$
          Where $SH\_C0$ is the first coefficient of the spherical harmonics basis (typically $\frac{1}{\sqrt{4\pi}}$) [S1, unofficial].
        - Else if `hasRGB` is true: Clamp `red_data[i]`, `green_data[i]`, `blue_data[i]` to $[0, 255]$ and assign to $r, g, b$ [S1, unofficial].
        - Else: Set $r = g = b = 255$ (white) [S1, unofficial].
    - **Opacity:**
        - If `opacity_data` is not null: Compute sigmoid.
          $$ a = \text{sigmoid}(opacity\_data[i]) = \frac{1.0f}{1.0f + \exp(-opacity\_data[i])} $$
          Clamp $a$ to $[0, 255]$ (interpreted as integer range for storage) and assign to $a$ [S1, unofficial].
        - Else: Set $a = 255$ (fully opaque) [G].
    - **Store:** Set `colors[i] = glm::u8vec4(r, g, b, a)` [S1, unofficial].

5.  **Packaging:**
    - Call `data.pack()` to prepare GPU data. This function iterates over all Gaussians to compute covariances and pack data into a contiguous buffer `packedData` of size `n * 8` uint32s [S1, unofficial].
    - For each Gaussian `i`:
        - Copy `positions[i]` bytes to `packedData[i*8 + 0..2]` using `memcpy` to avoid strict aliasing violations [S1, unofficial].
        - Set `packedData[i*8 + 3] = 0` (selection flag) [S1, unofficial].
        - Compute rotation matrix $M_{rot} = \text{mat3\_cast}(rotations[i])$ and scale matrix $M_{scale} = \text{diag}(scales[i])$ [S1, unofficial].
        - Compute transformation matrix $M = M_{rot} \times M_{scale}$ [S1, unofficial].
        - Compute symmetric covariance matrix elements $\sigma$:
          $$ \sigma_0 = M_{00}^2 + M_{10}^2 + M_{20}^2 \quad (\text{xx}) $$
          $$ \sigma_1 = M_{00}M_{01} + M_{10}M_{11} + M_{20}M_{21} \quad (\text{xy}) $$
          $$ \sigma_2 = M_{00}M_{02} + M_{10}M_{12} + M_{20}M_{22} \quad (\text{xz}) $$
          $$ \sigma_3 = M_{01}^2 + M_{11}^2 + M_{21}^2 \quad (\text{yy}) $$
          $$ \sigma_4 = M_{01}M_{02} + M_{11}M_{12} + M_{21}M_{22} \quad (\text{yz}) $$
          $$ \sigma_5 = M_{02}^2 + M_{12}^2 + M_{22}^2 \quad (\text{zz}) $$
        - Pack $\sigma$ into half-precision floats into `packedData[i*8 + 4..6]` using `packHalf2x16` [S1, unofficial].
        - Pack `colors[i]` (RGBA) into `packedData[i*8 + 7]` [S1, unofficial].

**Parameters:**
- $SH\_C0$: Spherical Harmonics DC coefficient constant. Value not explicitly given in sources, but implied by the formula structure [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Missing Properties:** If RGB or Opacity properties are missing, the loader must handle defaults (white color, opaque) without crashing [S1, unofficial].
- **Strict Aliasing:** Direct casting of float pointers to uint32 pointers for packing can violate strict aliasing rules; `memcpy` should be used for position copying [S1, unofficial].
- **Normalization:** Rotations must be normalized to unit quaternions to ensure valid rotation matrices [S1, unofficial].
- **Log-Space Scales:** Scales are stored in log-space to prevent negative values during optimization; they must be exponentiated upon loading [S1, unofficial].

## Subtopics

### Scene Storage Format Modification
This variant modifies the standard 3DGS PLY format to enable direct streaming to GPU memory, eliminating decoding overhead and optimizing for cache locality [S2]. It removes obsolete normal properties, reorders spherical harmonics data for better cache-locality, decodes opacity and scale properties during storage/streaming, and normalizes the rotation quaternion [S2]. It also limits the scene to a central cube, excluding Gaussians with means outside this volume [S2]. The mesh is stored alongside the modified scene file, including a list of page IDs (one entry per mesh face), a buffer containing page links, and metadata (page size, number of LOD levels) [S2].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of Gaussians $N$ for parsing and decoding $O(N)$. The packing step involves matrix multiplications per Gaussian, also $O(N)$.
- **Memory Complexity:** $O(N)$ for storing the decoded properties and the packed GPU buffer. The packed buffer uses 8 uint32s (32 bytes) per Gaussian [S1, unofficial].
- **Strengths:** Handles standard and modified PLY formats; decodes log-space scales and sigmoid opacity correctly; prepares data for efficient GPU rasterization.
- **Limitations:** Requires the PLY file to contain specific property names; does not handle coordinate system conversion (delegated to sibling method); does not handle compression (deferred to future work in modified format) [S2].
- **When to use:** When loading a pre-trained 3D Gaussian Splatting model from a PLY file for real-time rendering.

## Background
The method is based on the implementation in the `gsplat-opengl` repository, an unofficial OpenGL-based 3D Gaussian Splatting renderer [S1, unofficial]. The modifications for virtual memory streaming and cache locality are described in "Virtual memory for 3D Gaussian Splatting" [S2].

## Sources
[S1] liaozhangsheng/gsplat-opengl, https://github.com/liaozhangsheng/gsplat-opengl, fetched 2026-10-28 (unofficial)
[S2] Virtual memory for 3D Gaussian Splatting, https://zenodo.org/records/19690968, fetched 2026-10-26
