# Blinn-Phong Shading in Volume Rendering

Also called: Shaded Volume Rendering (SVR)

## Overview
Blinn-Phong Shading in Volume Rendering (SVR) is a technique that applies Blinn-Phong lighting models to volume data during ray marching to enhance depth perception and surface detail [S1, unofficial]. It differs from unshaded volume rendering by computing per-sample normals via central differences and combining them with diffuse, specular, and ambient terms, setting it apart from simpler shading models like Distance Color Blending or pure Phong shading with SDF normals by its specific implementation of the half-vector specular component and front-to-back compositing with lighting [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D volume texture (`texture_3d<f32>`), a 1D transfer function texture (`texture_1d<f32>`), a uniform transform matrix, and lighting parameters.
- **Output:** A 2D color image with shaded volume rendering.

**Data Structures:**
- **Uniforms Struct:**
  - `transform`: `mat4x4<f32>` (View/Projection matrix).
  - `lightPos`: `vec3<f32>` (Light position in world space).
  - `lightColour`: `vec3<f32>` (Light color intensity).
  - `diffuse`: `f32` (Diffuse lighting coefficient).
  - `specular`: `f32` (Specular lighting coefficient).
  - `ambient`: `f32` (Ambient lighting coefficient).
  - `tfSize`: `f32` (Size of the transfer function texture).
- **Textures:**
  - `volumeTexture`: Binding 0, `texture_3d<f32>`.
  - `volumeSampler`: Binding 1, sampler.
  - `tfTexture`: Binding 3, `texture_1d<f32>`.

**Numbered Steps:**

1.  **Vertex Shader Setup:** Generate a full-screen triangle using vertex indices to produce positions `(-1, 3)`, `(-1, -1)`, and `(3, -1)` in clip space [S1, unofficial]. This covers the screen more efficiently than two triangles [S1, unofficial].
2.  **Ray Generation:** For each fragment with screen coordinate `coord.xy`, compute the ray origin `pos` and direction `dir` by transforming screen coordinates:
    - `pos = transform * vec4(coord.xy, 0.0, 1.0)`
    - `dir = transform * vec4(0.0, 0.0, 1.0, 0.0)` [S1, unofficial]
3.  **Ray-Box Intersection:** Calculate the entry (`near`) and exit (`far`) distances of the ray with the volume's axis-aligned bounding box (AABB) defined by `bboxMin` (0,0,0) and `bboxMax` (volume dimensions) [S1, unofficial].
    - Compute `inv_dir = 1.0 / dir`.
    - Compute `bot = inv_dir * (bboxMin - pos)` and `top = inv_dir * (bboxMax - pos)`.
    - Compute `rmin = min(top, bot)` and `rmax = max(top, bot)`.
    - `t_entry = max(rmin.x, max(rmin.y, rmin.z))`
    - `t_exit = min(rmax.x, min(rmax.y, rmax.z))`
    - If `t_entry > t_exit`, the ray misses. Otherwise, `near = max(0.0, t_entry)` and `far = t_exit` [S1, unofficial].
4.  **Ray Marching Loop:** Iterate from `near` to `far` along the ray direction. For each step (or slice, depending on implementation granularity; the source iterates through slices using integer steps `k`):
    - Compute the current position `current_pos` in volume space.
    - Normalize coordinates to `[0, 1]` range for texture sampling: `coords = current_pos / volume_dimensions`.
    - Sample the volume texture: `sample = textureSample(volumeTexture, volumeSampler, coords)`.
    - Convert the RG8 format sample to a normalized 16-bit intensity value:
      $$ intensity = \frac{sample.x + sample.y \times 255}{256} $$
      [S1, unofficial]
    - **Normal Calculation:** Compute the gradient using central differences. For each axis $i \in \{x, y, z\}$:
      - Sample intensity at `pos + offset` and `pos - offset`.
      - Compute gradient component: $\delta_i = intensity(pos + offset_i) - intensity(pos - offset_i)$.
      - Normalize the resulting vector $\vec{\delta}$ to get the surface normal $\vec{n}$ [S1, unofficial].
    - **Transfer Function Lookup:** Map the sampled intensity `val` to a 1D texture index:
      $$ tfIndex = \lfloor val \times tfSize \rfloor $$
      Load the color from `tfTexture` at `tfIndex` to get `colour` [S1, unofficial].
    - **Blinn-Phong Lighting:**
      - Compute light direction $\vec{l}$, view direction $\vec{v}$, and half vector $\vec{h} = \frac{\vec{l} + \vec{v}}{|\vec{l} + \vec{v}|}$ [S1, unofficial].
      - Diffuse term: $diffuse\_term = \max(0.0, \vec{n} \cdot \vec{l}) \times diffuse$
      - Specular term: $specular\_term = pow(\max(0.0, \vec{n} \cdot \vec{h}), shininess) \times specular$
      - Ambient term: $ambient\_term = ambient$
      - Combined lit color:
        $$ litColor = lightColour \times (colour.rgb \times (diffuse\_term + ambient\_term) + specular\_term) $$
        [S1, unofficial]
    - **Front-to-Back Composition:** Update accumulated color `accColour` using alpha blending:
      $$ \alpha = (1.0 - accColour.a) \times colour.a $$
      $$ accColour.rgb += \alpha \times litColor.rgb $$
      $$ accColour.a += \alpha $$
      Break if `accColour.a >= 0.95` [S1, unofficial].

**Parameters:**
- `shininess`: Exponent for specular lighting, typically set to 10.0 [S1, unofficial].

**Edge Cases and Pitfalls:**
- Precision issues may arise when converting between integer slice indices and floating-point positions.
- Boundary conditions must be handled when sampling near the edges of the volume texture to avoid out-of-bounds access.
- The central difference normal calculation assumes smooth intensity gradients; noisy data may produce erratic normals [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of samples along the ray, which depends on the volume depth and step size.
- **Memory Complexity:** Constant additional memory for uniforms and accumulated color, plus storage for the volume and transfer function textures.
- **Performance:** Dependent on GPU capabilities and volume resolution. The use of a full-screen triangle and efficient ray-box intersection helps minimize overhead.
- **Strengths:** Provides realistic lighting effects, enhancing the perception of 3D structure.
- **Limitations:** Computationally more expensive than unshaded rendering due to normal calculation and lighting per sample. Sensitive to noise in volume data affecting normal estimation.

## Background
Blinn-Phong Shading in Volume Rendering is implemented in the `ossium` project, a WebGPU-based application for displaying 3D volumes created from DICOM files in the browser [S1, unofficial]. It was developed by Fraser Love as part of the ossium repository, aiming to provide interactive volume visualization with shading capabilities [S1, unofficial].

## Sources
[S1] fraserlove/ossium, https://github.com/fraserlove/ossium, fetched 2026-10-27 (unofficial)
