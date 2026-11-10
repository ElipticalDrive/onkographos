# Volume Raycasting with MRT Endpoints

Also called: MRT Ray Start/Stop Determination, Two-pass raycasting

## Overview
Volume Raycasting with MRT Endpoints is a two-pass GPU technique for determining the entry and exit points of rays into a volume by rendering the volume's bounding box to multiple render targets (MRT), followed by a second pass to sample the volume along those rays [S1, unofficial][S2, unofficial]. It builds on [Back-Face Culling for Ray Casting](1236-back-face-culling-for-ray-casting.md) by explicitly separating front-face and back-face coordinates into distinct textures to avoid analytical ray-cube intersection calculations [S1, unofficial][S2, unofficial]. Unlike single-pass methods that compute intersections in the fragment shader, this approach offloads geometry processing to the rasterizer, simplifying the raycasting shader at the cost of an additional rendering pass and texture lookups [S1, unofficial].

## How it works
**Inputs:**
- A 3D volume texture (typically single-channel density) [S1, unofficial][S2, unofficial].
- A model-view-projection matrix for the volume bounding box [S1, unofficial][S2, unofficial].
- A step length parameter defining sample spacing [S1, unofficial][S2, unofficial].
- Optional: A jitter texture for stochastic sampling [S1, unofficial].

**Outputs:**
- A 2D color buffer containing the final composited image [S1, unofficial][S2, unofficial].

**Data Structures:**
- Framebuffer Object (FBO) with two color attachments (`GL_COLOR_ATTACHMENT0` and `GL_COLOR_ATTACHMENT1`) [S1, unofficial][S2, unofficial].
- Two 2D floating-point RGB textures bound to the FBO attachments to store ray start and stop coordinates [S1, unofficial][S2, unofficial].
- 3D texture for volume data [S1, unofficial][S2, unofficial].

**Steps:**

1. **Pass 1: Compute Ray Endpoints**
   - Clear the two color attachments to black (zero) [S2, unofficial].
   - Enable blending with `GL_ONE, GL_ONE` (additive blending) [S1, unofficial][S2, unofficial].
   - Bind the FBO with two color attachments and set draw buffers:
     ```cpp
     GLenum renderTargets[2] = {GL_COLOR_ATTACHMENT0, GL_COLOR_ATTACHMENT1};
     glDrawBuffers(2, &renderTargets[0]);
     ```
   - Render a two-unit cube (vertices in range `[-1, 1]`) representing the volume bounding box [S1, unofficial][S2, unofficial].
   - In the vertex shader, apply the model-view-projection transform to `gl_Position` and pass the original object-space position `vPosition` to the fragment shader [S1, unofficial][S2, unofficial].
   - In the fragment shader, determine if the current face is front-facing using `gl_FrontFacing` [S1, unofficial][S2, unofficial].
     - If front-facing: write `0.5 * (vPosition + 1.0)` to `FragData[0]` (start point) and `vec3(0)` to `FragData[1]` (stop point) [S1, unofficial][S2, unofficial].
     - If back-facing: write `vec3(0)` to `FragData[0]` and `0.5 * (vPosition + 1.0)` to `FragData[1]` [S1, unofficial][S2, unofficial].
   - The additive blending ensures that only one value (start or stop) is written per pixel, as the other is zero [S1, unofficial][S2, unofficial].

2. **Pass 2: Volume Raycasting**
   - Render a full-screen quad covering the viewport [S1, unofficial][S2, unofficial].
   - Bind the two endpoint textures (RayStart, RayStop) and the volume texture [S1, unofficial][S2, unofficial].
   - For each fragment, sample the start and stop points using screen coordinates `coord = 0.5 * (vPosition.xy + 1.0)` [S1, unofficial][S2, unofficial].
   - If `rayStart == rayStop`, discard the fragment (no intersection) [S1, unofficial][S2, unofficial].
   - Compute the ray vector, length, and step vector:
     $$
     \text{ray} = \text{rayStop} - \text{rayStart}
     $$
     $$
     \text{rayLength} = \|\text{ray}\|
     $$
     $$
     \text{stepVector} = \text{StepLength} \times \frac{\text{ray}}{\text{rayLength}}
     $$
   - Apply stochastic jitter to the start position if a jitter texture is available:
     $$
     \text{rayStart} += \text{stepVector} \times \text{jitter}(\text{coord}).r
     $$
     [S1, unofficial]
   - Initialize `position = rayStart` and destination color `dst = vec4(0)` [S1, unofficial][S2, unofficial].
   - Loop while `rayLength > 0` (and optionally `dst.a < 1` for early termination) [S1, unofficial][S2, unofficial]:
     - Sample the volume: `density = texture(Volume, position).r` (or `.x`) [S1, unofficial][S2, unofficial].
     - Perform front-to-back blending:
       $$
       \text{src} = \text{vec4}(\text{density})
       $$
       $$
       \text{src}.rgb \times= \text{src}.a
       $$
       $$
       \text{dst} = (1.0 - \text{dst}.a) \times \text{src} + \text{dst}
       $$
       [S2, unofficial]
     - Advance position: `position += stepVector` [S1, unofficial][S2, unofficial].
     - Decrement length: `rayLength -= StepLength` [S1, unofficial][S2, unofficial].
   - Output `dst` as the fragment color [S2, unofficial].

**Parameters:**
- `StepLength`: Distance between samples. Typical value: 0.01 [S2, unofficial]. Balances quality and speed; larger values are faster but lower quality, smaller values are slower but higher quality [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Non-intersecting rays:** Handled by discarding fragments where `rayStart == rayStop` [S1, unofficial][S2, unofficial].
- **Jitter:** Stochastic jitter helps reduce aliasing artifacts (e.g., wood grain) but requires a pre-computed jitter texture [S1, unofficial].

**Pseudocode (Fragment Shader for Pass 2):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def raycast_pass2(ray_start_tex, ray_stop_tex, volume_tex, step_length, jitter_tex=None):
    # Screen coordinate conversion
    coord = 0.5 * (v_position.xy + 1.0)
    
    # Sample endpoints
    ray_start = ray_start_tex.sample(coord).xyz
    ray_stop = ray_stop_tex.sample(coord).xyz
    
    # Discard non-intersecting rays
    if ray_start == ray_stop:
        return  # Fragment discarded
    
    # Compute ray properties
    ray = ray_stop - ray_start
    ray_length = length(ray)
    step_vector = step_length * ray / ray_length
    
    # Optional stochastic jitter
    if jitter_tex is not None:
        ray_start += step_vector * jitter_tex.sample(coord).r
        
    pos = ray_start
    dst = vec4(0.0)
    
    # Raymarching loop
    while ray_length > 0 and dst.a < 1.0:
        density = volume_tex.sample(pos).r
        src = vec4(density)
        src.rgb *= src.a
        dst = (1.0 - dst.a) * src + dst
        pos += step_vector
        ray_length -= step_length
        
    return dst
```

## Subtopics

### Volumetric Lighting via Gradient Normals
A variant that computes lighting normals in volume data by approximating the gradient vector, which points in the direction of greatest change and whose magnitude indicates surface intersection [S2, unofficial]. It computes the gradient at position `P` using central differences with step size `L`: `grad.x = texture(P + vec3(L,0,0)) - texture(P)`, `grad.y = texture(P + vec3(0,L,0)) - texture(P)`, `grad.z = texture(P + vec3(0,0,L)) - texture(P)` [S2, unofficial]. The gradient is only computed if the current voxel value `V` is above a `Threshold` (typical value: 0.45) to avoid excessive texture lookups [S2, unofficial]. When the threshold is met, the gradient is normalized, diffuse lighting is computed (`df = abs(dot(normal, light))`, `color = df * DiffuseMaterial`), `src = vec4(color, 1.0)`, front-to-back blending is applied, and the loop breaks [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Two rendering passes. The second pass performs one texture lookup per sample along the ray. Total work is proportional to the number of pixels times the average ray length in samples [S1, unofficial].
- **Memory Complexity:** Requires two additional 2D textures for ray endpoints, plus the volume texture [S1, unofficial][S2, unofficial].
- **Performance:** Simpler fragment shader than analytical intersection methods, but incurs overhead from the first pass and two extra texture samples in the second pass (RayStart, RayStop) [S1, unofficial].
- **Strengths:** Easy to implement; leverages GPU rasterization for intersection; supports early termination with front-to-back blending [S1, unofficial][S2, unofficial].
- **Limitations:** Two-pass overhead; extra texture lookups make it suboptimal for performance-critical applications; requires MRT support [S1, unofficial][S2, unofficial].
- **When to use:** Simple volume rendering where bounding box is axis-aligned or easily transformed; when implementation simplicity is prioritized over performance [S1, unofficial].
- **When not to use:** When single-pass performance is critical due to pass overhead and texture sampling cost [S1, unofficial].

## Background
Originated as a simple technique for volumetric raycasting on the GPU, described in blog posts and tutorials around 2018 [S1, unofficial][S2, unofficial]. It is often referred to as "two-pass raycasting" due to its structure [S1, unofficial]. The method is used in educational contexts and simple volume renderers where ease of implementation is key [S1, unofficial][S2, unofficial].

## Sources
[S1] GPU-accelerated single-pass volumetric raycasting in Qt ..., https://martinopilia.com/posts/2018/09/17/volume-raycasting.html, fetched 2026-10-27 (unofficial)
[S2] Volumetric Splatting at The Little Grasshopper, https://prideout.net/blog/old/blog/index.html%40p%3D60.html, fetched 2026-10-28 (unofficial)
