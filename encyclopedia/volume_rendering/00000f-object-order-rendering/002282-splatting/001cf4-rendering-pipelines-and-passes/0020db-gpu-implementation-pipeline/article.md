# GPU Implementation Pipeline
Also called: Single-pass GPU Raycasting, Volumetric Raycasting

## Overview
A GPU-accelerated rendering technique that traces rays from the camera through each pixel into a 3D volume to compute color and opacity, implemented in a single pass using fragment shaders [S2, unofficial]. It solves the problem of visualizing volumetric data directly on the GPU without generating intermediate polygonal meshes. It differs from isosurface extraction (e.g., [Marching Cubes](53-marching-cubes.md)) by preserving internal volume information and from splatting ([Splatting](10-splatting.md)) by avoiding sorting and blending overheads for dense volumes, though it is computationally heavier per pixel than texture-based splatting.

## How it works
**Inputs:**
- 3D volume texture (scalar field, typically `GL_RED` format).
- Camera parameters: View Matrix, Model-View-Projection (MVP) Matrix, Ray Origin, Ray Direction (derived from screen coordinates).
- Rendering parameters: Step length, bounding box (top/bottom), background color, gamma, transfer function coefficients.
- Optional: Jitter texture for anti-aliasing.

**Outputs:**
- Final color buffer (RGBA).

**Data Structures:**
- **Volume Texture:** 3D texture object storing scalar intensity values. In multi-resolution variants, this may be a reduced structure (e.g., $256 \times 256 \times 128$ for a $512^3$ dataset) [S1, unofficial].
- **Bounding Box (AABB):** Defined by `top` and `bottom` vectors in texture coordinates or world space.
- **Ray Structure:** Contains `origin` (vec3) and `direction` (vec3).

**Numbered Steps:**

1. **Initialization:**
   - Create OpenGL state and volume data structure [S2, unofficial].
   - Compute the view matrix by translating along the Z-axis: $T_z = -4.0 \cdot \exp(\frac{distExp}{600.0})$ [S2, unofficial].
   - Compute the MVP matrix using perspective projection with FOV ($m_{fov}$), aspect ratio ($\frac{scaled\_width}{scaled\_height}$), near plane $0.1$, and far plane $100.0$ [S2, unofficial].
   - Compute ray origin in world space by inverting the view matrix and transforming $(0,0,0)$ [S2, unofficial].
   - Bind shader program and set uniforms: `ViewMatrix`, `ModelViewProjectionMatrix`, `aspect_ratio`, `focal_length`, `viewport_size`, `ray_origin`, `top`, `bottom`, `background_colour`, `step_length`, `gamma`, `volume` (texture unit 0), `jitter` (texture unit 1) [S2, unofficial].
   - Clear color buffer with `background_colour` [S2, unofficial].

2. **Ray Generation (Vertex/Fragment Shader Entry):**
   - For each fragment (pixel), compute normalized device coordinates (NDC) from `gl_FragCoord`.
   - Compute ray direction in view space:
     $$
     ray\_dir_{xy} = 2.0 \cdot \frac{gl\_FragCoord_{xy}}{viewport\_size} - 1.0
     $$
     $$
     ray\_dir_x \cdot aspect\_ratio
     $$
     $$
     ray\_dir_z = -focal\_length
     $$
   - Transform ray direction to world space:
     $$
     ray\_direction = (ViewMatrix \cdot vec4(ray\_dir, 0.0)).xyz
     $$

3. **Ray-Box Intersection:**
   - Perform slab method intersection between `casting_ray` and `AABB` (defined by `top` and `bottom`) to find entry ($t_0$) and exit ($t_1$) distances [S2, unofficial].
   - Formula for intersection:
     $$
     direction\_inv = \frac{1.0}{ray.direction}
     $$
     $$
     t_{top} = direction\_inv \cdot (box.top - ray.origin)
     $$
     $$
     t_{bottom} = direction\_inv \cdot (box.bottom - ray.origin)
     $$
     $$
     t_{min} = min(t_{top}, t_{bottom})
     $$
     $$
     t_0 = max(0.0, max(t_{min}.x, t_{min}.y))
     $$
     $$
     t_1 = min(t_{max}.x, t_{max}.y) \quad \text{where } t_{max} = max(t_{top}, t_{bottom})
     $$

4. **Ray Marching Setup:**
   - Convert world space intersection points to texture coordinates:
     $$
     ray\_start = \frac{ray.origin + ray.direction \cdot t_0 - bottom}{top - bottom}
     $$
     $$
     ray\_stop = \frac{ray.origin + ray.direction \cdot t_1 - bottom}{top - bottom}
     $$
   - Compute ray vector and length:
     $$
     ray = ray\_stop - ray\_start
     $$
     $$
     ray\_length = length(ray)
     $$
   - Compute step vector:
     $$
     step\_vector = step\_length \cdot \frac{ray}{ray\_length}
     $$
   - Apply random jitter to start position using a jitter texture:
     $$
     ray\_start += step\_vector \cdot texture(jitter, \frac{gl\_FragCoord}{viewport\_size}).r
     $$
   - Initialize current position: $position = ray\_start$ [S2, unofficial].

5. **Marching Loop (Front-to-Back Alpha Blending):**
   - While $ray\_length > 0$ and accumulated opacity $colour.a < 1.0$:
     - Sample volume intensity: $intensity = texture(volume, position).r$ [S2, unofficial].
     - Apply Transfer Function (TF) to get color $c$ and opacity $\alpha_c$:
       - Simple exponential TF example:
         $$
         \alpha_c = \frac{\exp(intensity) - 1.0}{\exp(1.0) - 1.0}
         $$
         $$
         c.rgb = intensity \cdot high + (1.0 - intensity) \cdot low
         $$
         [S2, unofficial]
     - Composite:
       $$
       colour.rgb = c.a \cdot c.rgb + (1 - c.a) \cdot colour.a \cdot colour.rgb
       $$
       $$
       colour.a = c.a + (1 - c.a) \cdot colour.a
       $$
     - Advance:
       $$
       ray\_length -= step\_length
       $$
       $$
       position += step\_vector
       $$
   - [G] Ensure `position` remains within texture bounds to prevent out-of-bounds sampling artifacts.

6. **Final Compositing:**
   - Blend with background:
     $$
     colour.rgb = colour.a \cdot colour.rgb + (1 - colour.a) \cdot pow(background\_colour, vec3(gamma)).rgb
     $$
     $$
     colour.a = 1.0
     $$
   - Apply gamma correction to output:
     $$
     output.rgb = pow(colour.rgb, vec3(\frac{1.0}{gamma}))
     $$
     [S2, unofficial]

**Parameters:**
- `step_length`: Controls sampling density. Typical values depend on volume resolution and desired quality. In multi-resolution variants, piecewise segment lengths of 8-16 voxels provide significant performance gains [S1, unofficial].
- `gamma`: Gamma correction value (e.g., 2.2).
- `focal_length`: Determines field of view.
- `distExp`: Exponential distance exponent for camera translation [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Empty Blocks:** In multi-resolution structures, if a sample point falls in an empty block, the ray must be advanced to the next non-empty block to avoid losing occlusion from neighboring blocks [S1, unofficial].
- **Interpolation Artifacts:** Flat blocking multi-resolution data management can cause interpolation problems if blocks are not handled correctly [S1, unofficial].
- **Precision:** Ray-box intersection precision affects entry/exit points, especially for grazing angles.

## Subtopics

### [Volume Rendering with 3D Textures and Ray Marching](2584-volume-rendering-with-3d-textures-and-ray-marching.md)
This method performs object-order volume rendering by ray marching through a 3D texture stored in GPU memory, applying a transfer function to map scalar values to color and opacity, and compositing the result front-to-back. It solves the problem of visualizing volumetric data (such as medical DICOM series or synthetic density fields) directly on the GPU without generating intermediate polygonal meshes. It differs from isosurface extraction (e.g., [Marching Cubes](53-marching-cubes.md)) by preserving internal volume information and from splatting ([Splatting](10-splatting.md)) by avoiding sorting and blending overheads for dense volumes, though it is computationally heavier per pixel than texture-based splatting.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of samples per ray and the number of pixels. Performance is heavily influenced by the step length and volume resolution.
- **Memory Complexity:** Proportional to the size of the 3D volume texture. Multi-resolution structures can reduce storage by factors of 8 or more (e.g., $256^3$ for a $512^3$ dataset) [S1, unofficial].
- **Measured Performance:** On an Nvidia GeForce 8800 Ultra with 768 MB graphics texture memory, for a $512^3$ volume with 8.9:1 data reduction and SVR size $256^3$:
  - Global light integration (A) takes 284-552 ms [S1, unofficial].
  - First order scattering incremental refinements (B) take 62-233 ms [S1, unofficial].
  - High frame rates (15-22 fps) are reached after light updates [S1, unofficial].
- **Strengths:** Single-pass implementation allows for real-time interaction. Simple to implement on modern GPUs. Preserves full volumetric information.
- **Limitations:** Computationally intensive per pixel compared to splatting. Performance degrades with large step sizes or high-resolution volumes without multi-resolution acceleration. Artifacts can occur with improper handling of empty blocks in multi-resolution structures [S1, unofficial].
- **When to use:** Real-time visualization of dense volumetric data where internal structure is important.
- **When not to use:** When extreme performance is required for very large volumes without multi-resolution support, or when surface-only visualization is sufficient (use Marching Cubes).

## Background
The technique is based on GPU-accelerated single-pass volumetric raycasting, implemented in GLSL and C++ with the Qt framework [S2, unofficial]. It aims to provide a basic skeleton for a visualiser that can be easily extended with a feature-rich GUI [S2, unofficial]. The approach of using multi-resolution data structures and piecewise integration for global light propagation was introduced in "Interactive Global Light Propagation in Direct Volume Rendering using Local Piecewise Integration" [S1, unofficial]. This work demonstrated that reducing the volume size via flat blocking multi-resolution data management allows for significant performance improvements without significant loss in visual quality [S1, unofficial].

## Sources
[S1] Interactive Global Light Propagation in Direct Volume Rendering using Local Piecewise Integration, https://doi.org/10.2312/vg/vg-pbg08/105-112, fetched 2026-10-26 (unofficial)
[S2] GitHub - m-pilia/volume-raycasting: Simple visualiser based on GPU ..., https://github.com/m-pilia/volume-raycasting, fetched 2026-10-27 (unofficial)
