# Blinn-Phong Shading in Volume Rendering

Also called: Shaded Volume Rendering (SVR)

## Overview
Blinn-Phong Shading in Volume Rendering is a technique that applies the Blinn-Phong lighting model to volume data during ray marching to produce shaded, three-dimensional appearances [S1, unofficial]. It distinguishes itself from unshaded methods (like MPR) by calculating surface normals via volume gradients and combining diffuse, specular, and ambient lighting components, while differing from Phong shading by using the half-vector for specular highlights rather than the reflection vector [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D volume texture (`volumeTexture`), a 1D transfer function texture (`tfTexture`), a transformation matrix (`transform`), and lighting parameters (`lightPos`, `lightColour`, `diffuse`, `specular`, `ambient`, `tfSize`) [S1, unofficial].
- **Output:** A 2D color image with alpha blending [S1, unofficial].

**Data Structures:**
- **Uniforms Struct:**
  - `transform`: `mat4x4<f32>` (4x4 matrix) [S1, unofficial].
  - `lightPos`: `vec3<f32>` [S1, unofficial].
  - `lightColour`: `vec3<f32>` [S1, unofficial].
  - `diffuse`: `f32` [S1, unofficial].
  - `specular`: `f32` [S1, unofficial].
  - `ambient`: `f32` [S1, unofficial].
  - `tfSize`: `f32` [S1, unofficial].
- **Textures:**
  - `volumeTexture`: `texture_3d<f32>` at binding 0 [S1, unofficial].
  - `volumeSampler`: sampler at binding 1 [S1, unofficial].
  - `tfTexture`: `texture_1d<f32>` at binding 3 [S1, unofficial].

**Steps:**

1.  **Vertex Shader Setup:** Generate a full-screen triangle using vertex indices to produce clip-space positions: $(-1, 3)$, $(-1, -1)$, and $(3, -1)$ [S1, unofficial].
2.  **Ray Generation:** For each fragment with screen coordinate $coord.xy$, compute the ray origin $pos$ and direction $dir$ in volume space:
    $$ pos = transform \times vec4(coord.xy, 0.0, 1.0) $$
    $$ dir = transform \times vec4(0.0, 0.0, 1.0, 0.0) $$
    [S1, unofficial]
3.  **Ray-Box Intersection:** Calculate entry ($near$) and exit ($far$) distances by intersecting the ray with the volume's axis-aligned bounding box (AABB).
    - Compute inverse direction: $inv\_dir = 1.0 / dir$ [S1, unofficial].
    - Compute distances to bounds: $bot = inv\_dir \times (bboxMin - pos)$, $top = inv\_dir \times (bboxMax - pos)$ [S1, unofficial].
    - Compute min/max intersections: $rmin = min(top, bot)$, $rmax = max(top, bot)$ [S1, unofficial].
    - Entry: $near = max(0.0, max(rmin.x, rmin.y, rmin.z))$ [S1, unofficial].
    - Exit: $far = min(rmax.x, rmax.y, rmax.z)$ [S1, unofficial].
    - If $near > far$, the ray misses the volume [S1, unofficial].
4.  **Ray Marching Loop:** Iterate from $near$ to $far$ along the ray.
    - For each step $k$ (representing slice index or distance):
        - Compute current position: $transformed = transform \times vec4(coord.xy, k, 1.0)$ [S1, unofficial].
        - Normalize coordinates: $coords = transformed.xyz / size$ [S1, unofficial].
        - Sample volume: $sample = textureSample(volumeTexture, volumeSampler, coords)$ [S1, unofficial].
        - Convert intensity: $val = (sample.x + sample.y \times 255) / 256$ [S1, unofficial].
        - Calculate Normal: Compute gradient using central differences.
          - Sample neighbors at $pos \pm offset$ for each axis.
          - $delta = normalize(sample(pos+offset) - sample(pos-offset))$ [S1, unofficial].
        - Transfer Function Lookup: Map intensity to color.
          - $tfIndex = val \times tfSize$ [S1, unofficial].
          - $colour = textureSample(tfTexture, sampler, tfIndex)$ [S1, unofficial].
        - Blinn-Phong Lighting:
          - Light direction: $lightDir = normalize(lightPos - pos)$ [S1, unofficial].
          - View direction: $viewDir = normalize(-pos)$ [S1, unofficial].
          - Half vector: $halfDir = normalize(lightDir + viewDir)$ [S1, unofficial].
          - Diffuse: $diff = max(0.0, dot(normal, lightDir)) \times diffuse$ [S1, unofficial].
          - Specular: $spec = pow(max(0.0, dot(normal, halfDir)), shininess) \times specular$ [S1, unofficial].
          - Ambient: $amb = ambient$ [S1, unofficial].
          - Combined Color: $litColor = lightColour \times (colour.rgb \times (diff + amb) + spec)$ [S1, unofficial].
        - Front-to-Back Composition:
          - $\alpha = (1.0 - accColour.a) \times colour.a$ [S1, unofficial].
          - $accColour.rgb += \alpha \times colour.rgb$ [S1, unofficial].
          - $accColour.a += \alpha$ [S1, unofficial].
          - Break if $accColour.a \ge 0.95$ [S1, unofficial].

**Parameters:**
- `shininess`: Exponent for specular calculation, typically 10.0 [S1, unofficial].

**Edge Cases and Pitfalls:**
- Precision issues may arise when converting 8-bit packed samples to 16-bit normalized values [S1, unofficial].
- Ray-box intersection must handle cases where the ray origin is inside the volume ($near < 0$) by clamping to 0 [S1, unofficial].
- Degenerate normals (zero-length gradients) should be handled by normalizing only if the length is non-zero [G].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of samples along the ray, which depends on volume resolution and view angle [S1, unofficial].
- **Memory Complexity:** Proportional to the size of the 3D volume texture and 1D transfer function texture [S1, unofficial].
- **Performance:** Efficient for real-time rendering on WebGPU-enabled browsers due to the use of full-screen triangles and early termination based on alpha accumulation [S1, unofficial].
- **Strengths:** Provides realistic shading and depth cues through specular highlights and diffuse lighting [S1, unofficial].
- **Limitations:** Computationally more expensive than unshaded methods due to normal calculation and lighting per sample [S1, unofficial].

## Background
Blinn-Phong Shading in Volume Rendering was implemented in the `ossium` project, a WebGPU-based volume rendering application for displaying DICOM files in the browser [S1, unofficial]. It serves as an alternative to Multi-Planar Reformatting (MPR) by adding shading to enhance the perception of 3D structure [S1, unofficial].

## Sources
[S1] fraserlove/ossium, https://github.com/fraserlove/ossium, fetched 2026-10-27 (unofficial)
