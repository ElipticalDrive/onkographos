# Ray Marching Dither

Also called: UV Perturbation Dither, Distance-Based UV Dither, Ray Marching Jittering (step variant), Blue Noise Dithering (texture variant)

## Overview
Ray Marching Dither is a family of techniques used in distance-field ray marching to suppress banding and layering artifacts caused by low step counts or discrete sampling intervals [S1, S2, S3, unofficial]. Unlike standard fixed-step marching which produces visible intensity bands, these methods introduce controlled randomness to the sampling process—either by perturbing the ray's UV coordinates based on hit distance [S1, unofficial], offsetting the initial depth using a blue noise texture [S2, unofficial], or jittering the step size itself [S3, unofficial]. This breaks up coherent aliasing patterns while maintaining the sharpness of volumetric features like god rays [S3, unofficial].

## How it works
The method varies by variant, but all aim to decorrelate the sampling grid from the visual output. The following sections detail the three primary implementations found in the sources.

### Distance-Based UV Perturbation
This variant perturbs the texture coordinates used to reconstruct the ray origin and direction, effectively shifting the ray path based on the accumulated distance traveled [S1, unofficial].

**Inputs and Outputs:**
- **Inputs:** Fragment UV coordinates (`vUv`), hit distance (`t`), dither strength factor (`c_fDither`).
- **Outputs:** Perturbed UV coordinates (`uv`), reconstructed ray origin (`ro`) and direction (`rd`).

**Data Structures:**
- `vUv`: 2D vector of fragment coordinates.
- `t`: Scalar float representing the ray hit distance.
- `c_fDither`: Scalar float parameter.

**Steps:**
1. Calculate the dither magnitude based on the squared hit distance and the strength factor:
   $$ dither = t^2 \cdot c\_fDither $$
2. Generate two random values using a pseudo-random function `rand` seeded by the fragment UV plus fixed offsets. The offsets are $(0.12, 0.32)$ for the X component and $(0.42, 0.52)$ for the Y component [S1, unofficial].
3. Perturb the UV coordinates:
   $$ uv.x = vUv.x + rand(vUv + \vec{2}(0.12, 0.32)) \cdot dither - \frac{dither}{2.0} $$
   $$ uv.y = vUv.y + rand(vUv + \vec{2}(0.42, 0.52)) \cdot dither - \frac{dither}{2.0} $$
4. Reconstruct the ray origin (`ro`) and direction (`rd`) using the perturbed `uv` values and the camera setup (camera position, up vector, aspect ratio, and field of view) [S1, unofficial].

**Parameters:**
- `c_fDither`: Dithering strength factor. Typical value: $0.001$ [S1, unofficial].

**Pitfalls:**
- The perturbation is distance-dependent ($t^2$), meaning artifacts further from the camera are dithered more aggressively. This may cause visual inconsistency if the camera moves rapidly [S1, unofficial].

### Blue Noise Dithering
This variant uses a pre-computed blue noise texture to offset the starting depth of the raymarch loop, reducing layering artifacts in volumetric scenes [S2, unofficial].

**Inputs and Outputs:**
- **Inputs:** Fragment coordinates (`gl_FragCoord`), frame count (`uFrame`), blue noise texture sampler (`uBlueNoise`), march step size (`MARCH_SIZE`).
- **Outputs:** Raymarched result with offset initial depth.

**Data Structures:**
- `uBlueNoise`: 2D sampler texture containing a blue noise pattern.
- `gl_FragCoord`: Built-in fragment coordinate vector.

**Steps:**
1. Sample the blue noise texture at normalized fragment coordinates:
   $$ \text{noise\_coord} = \frac{gl\_FragCoord.xy}{1024.0} $$
   $$ \text{blueNoise} = \text{texture2D}(uBlueNoise, \text{noise\_coord}).r $$
2. Compute the fractional offset:
   $$ \text{offset} = \text{fract}(\text{blueNoise}) $$
3. Add the offset scaled by the march step size to the initial depth before entering the raymarch loop:
   $$ \text{depth} = \text{MARCH\_SIZE} \cdot \text{offset} $$
   $$ \mathbf{p} = \mathbf{rayOrigin} + \text{depth} \cdot \mathbf{rayDirection} $$
4. **Temporal Variant:** To prevent the dithering pattern from being noticeable in single frames, add a frame-dependent value to the blue noise before taking the fractional part [S2, unofficial]:
   $$ \text{offset} = \text{fract}(\text{blueNoise} + \frac{\text{float}(uFrame \% 32)}{\sqrt{0.5}}) $$

**Parameters:**
- `MARCH_SIZE`: The fixed step size for the raymarch loop.
- `uBlueNoise`: Sampler for the 2D blue noise texture.

**Pitfalls:**
- Single-frame renders may exhibit visible noise patterns. Mitigation requires the temporal component described in Step 4 [S2, unofficial].
- Bicubic filtering is recommended for the blue noise texture to fix remaining noise artifacts [S2, unofficial].

### Ray Marching Jittering (Step Size Variant)
This variant introduces random variation directly to the step size during the march, countering aliasing in participating media [S3, unofficial].

**Inputs and Outputs:**
- **Inputs:** Fixed step size, random number generator.
- **Outputs:** Ray position with jittered step advancement.

**Steps:**
1. During the raymarch loop, instead of advancing by a fixed step size, march the ray a random distance:
   $$ \Delta t \in [0.5 \cdot \text{step\_size}, 1.5 \cdot \text{step\_size}] $$
2. Apply this jittered step to both the photon gathering and lookup stages [S3, unofficial].

**Pitfalls:**
- Fixed step sizes introduce clearly defined bands of intensity in volume regions [S3, unofficial].
- Jittering removes banding while keeping god rays sharp and focused, unlike fixed stepping [S3, unofficial].

## Subtopics

### Blue Noise Dithering
A technique that uses a blue noise texture to offset the sampling start point along the ray, reducing banding and layering artifacts caused by low step counts [S2, unofficial]. It is chosen when volumetric layering is prominent and temporal coherence can be maintained via frame-dependent offsets. It differs from UV perturbation by operating in depth space rather than UV space [S2, unofficial].

### Ray Marching Jittering
A technique that adds random variation to the ray marching step sizes (specifically between 0.5x and 1.5x the fixed step size) to counteract aliasing artifacts in volumes [S3, unofficial]. It is chosen for participating media where fixed stepping causes visible intensity bands. It differs from texture-based dithering by modifying the integration step directly rather than the initial conditions or UVs [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ overhead per fragment for UV perturbation [S1, unofficial] and Blue Noise sampling [S2, unofficial]. Step jittering [S3, unofficial] adds negligible overhead to the existing loop.
- **Memory Complexity:** Blue Noise dithering requires storing a 2D texture (e.g., 1024x1024 or similar, normalized by 1024.0 in [S2, unofficial]). UV perturbation requires no extra memory [S1, unofficial].
- **Performance:** All variants allow for lower step counts while maintaining visual quality, effectively trading a small amount of randomness for reduced computational cost or improved visual fidelity [S2, S3, unofficial].
- **Strengths:** Effectively removes banding and layering artifacts [S1, S2, S3, unofficial]. Keeps volumetric features like god rays sharp [S3, unofficial].
- **Limitations:** Blue noise patterns can be noticeable in single frames without temporal variation [S2, unofficial]. UV perturbation is distance-dependent, which may cause inconsistent dithering strength across the scene [S1, unofficial].

## Background
Ray marching dithering techniques emerged as practical solutions for real-time ray marching in WebGL and GPU-based rendering, where step counts are limited by performance constraints. The UV perturbation method is documented in the `WebGL-RayMarch` repository [S1, unofficial]. Blue noise dithering for cloudscapes was popularized in blog posts discussing real-time volumetric rendering [S2, unofficial]. Step jittering was employed in academic projects for rendering underwater scenes and participating media [S3, unofficial].

## Sources
[S1] WebGL-RayMarch, https://github.com/nopjia/WebGL-RayMarch, fetched 2026-10-26 (unofficial)
[S2] Real-time dreamy Cloudscapes with Volumetric Raymarching, https://blog.maximeheckel.com/posts/real-time-cloudscapes-with-volumetric-raymarching/, fetched 2026-10-28 (unofficial)
[S3] CS348b Final Project - Rendering Underwater Scenes, http://www.graphics.stanford.edu/courses/cs348b-competition/cs348b-04/underwater/index.html, fetched 2026-10-30 (unofficial)
