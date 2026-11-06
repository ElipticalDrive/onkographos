# Hybrid Gradient Volume Raycasting with Lasso Masking

Also called: Colored Raycast Volume Render, HU Direction + Opacity Threshold Lighting

## Overview
This method is a direct volume raycasting technique that renders medical CT data by combining pre-computed Hounsfield Unit (HU) gradients for lighting direction with a hybrid shading model that applies Phong lighting only when significant opacity changes are detected [S1, unofficial]. It incorporates lasso-based voxel masking to selectively hide or reveal regions of the volume and uses cutting cube bounds to define the rendering region [S1, unofficial]. It differs from standard direct volume rendering by decoupling the lighting direction from the sample gradient, using pre-computed gradients for stability, and by introducing conditional shading based on opacity thresholds to prevent artifacts on smooth structures like vessels [S1, unofficial].

## How it works
The method is implemented as a fragment shader pipeline that performs ray marching within a defined 3D bounding box. It relies on the parent's principles of front-to-back compositing and ray discretization [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   `volumeTexture`: `texture_2d_array<f32>` containing the scalar density values (HU).
    *   `gradientTexture`: `texture_2d_array<f32>` containing pre-computed gradient vectors.
    *   `ctfTexture`: `texture_1d<f32>` (or `vec4` texture) storing the Color Transfer Function (CTF) lookup table.
    *   `lassoMask`: `texture_3d<u32>` where voxels inside the lasso contour have a value of 0 and outside have a non-zero value (or vice versa, see Lasso Mask Check).
    *   `Uniforms`: A struct containing transformation matrices, camera position, volume dimensions, step size, and toggle flags.
*   **Outputs:**
    *   A `vec4<f32>` containing the accumulated color and opacity for the pixel.

### Data Structures
**Uniforms Struct Layout** [S1, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```wgsl
struct Uniforms {
  modelMatrix: mat4x4<f32>,
  viewMatrix: mat4x4<f32>,
  projectionMatrix: mat4x4<f32>,
  cameraPosition: vec3<f32>,
  _padding0: f32,
  numSlices: f32,
  stepSize: f32,
  densityScale: f32,
  _padding1: f32,
  inverseModelMatrix: mat4x4<f32>,
  _padding2: f32,
  _padding3: f32,
  _padding4: f32,
  _padding5: f32,
  voxelSpacing: vec3<f32>,
  _padding6: f32,
  toggleGradient: u32,
  volumeWidth: u32,
  volumeHeight: u32,
  volumeDepth: u32,
  xmin: f32,
  xmax: f32,
  ymin: f32,
  ymax: f32,
  zmin: f32,
  zmax: f32,
}
```
**Bindings** [S1, unofficial]:
*   `@group(0) @binding(0) uniforms: Uniforms`
*   `@group(0) @binding(1) volumeTexture: texture_2d_array<f32>`
*   `@group(0) @binding(2) volumeSampler: sampler`
*   `@group(0) @binding(3) gradientTexture: texture_2d_array<f32>`
*   `@group(0) @binding(4) ctfTexture: texture_1d<f32>`
*   `@group(0) @binding(5) lassoMask: texture_3d<u32>`

### Parameters
*   `stepSize`: `f32`, distance between samples along the ray [S1, unofficial].
*   `densityScale`: `f32`, scaling factor for density values [S1, unofficial].
*   `voxelSpacing`: `vec3<f32>`, physical spacing between voxels [S1, unofficial].
*   `toggleGradient`: `u32`, flag to enable/disable gradient-based lighting (0 or non-zero) [S1, unofficial].
*   `volumeWidth`, `volumeHeight`, `volumeDepth`: `u32`, dimensions of the volume texture [S1, unofficial].
*   `numSlices`: `f32`, number of slices in the volume texture array [S1, unofficial].

### Algorithm Steps
1.  **Vertex Shader**: Transform the input quad position to clip space using `modelMatrix`, `viewMatrix`, and `projectionMatrix`. Pass the world position and a zero ray direction to the fragment shader [S1, unofficial].
2.  **Fragment Shader Initialization**:
    *   Compute the camera position in cube space using the `inverseModelMatrix` [S1, unofficial].
    *   Set the ray origin to the camera position [S1, unofficial].
    *   Compute the ray direction by normalizing `(worldPosition - cameraPosition)` and scaling by `voxelSpacing` [S1, unofficial].
3.  **Ray-Box Intersection**:
    *   Intersect the ray with the box defined by `boxMin = -1.0/voxelSpacing` and `boxMax = 1.0/voxelSpacing` (or the cutting cube bounds `xmin/xmax/ymin/ymax/zmin/zmax`) [S1, unofficial].
    *   Return `tNear` and `tFar`. Discard the pixel if `tNear >= tFar` or `tFar <= 0` [S1, unofficial].
4.  **Jitter**:
    *   Compute a jitter offset to prevent banding artifacts:
        $$ jitter = fract(\sin(dot(input.clipPosition.xy, vec2(12.9898, 78.233))) \times 43758.5453) \times uniforms.stepSize $$
    *   Initialize the ray parameter $t$ with this jitter [S1, unofficial].
5.  **Ray Marching Loop**:
    *   While `accumulatedAlpha < 0.95` and $t < t_{Far}$:
        a.  **Position Conversion**:
            Compute the scaled position: $scaledPos = scaledRayOrigin + rayDir \times t$.
            Convert to texture space $[0, 1]$:
            $$ pos = scaledPos \times uniforms.voxelSpacing \times 0.5 + vec3(0.5) $$
            [S1, unofficial]
        b.  **Texture Bounds Check**:
            If `pos.x`, `pos.y`, or `pos.z` is outside $[0, 1]$, break the loop [S1, unofficial].
        c.  **Lasso Mask Check**:
            Compute voxel coordinates:
            $$ voxelCoord = (u32(pos.x \times (volumeWidth-1)), u32(pos.y \times (volumeHeight-1)), u32(pos.z \times (volumeDepth-1))) $$
            Load `maskValue` from `lassoMask`. If `maskValue == 0` (inside lasso contour), increment $t$ and `stepCount`, then `continue` (skip this sample) [S1, unofficial].
        d.  **Volume Sampling**:
            Sample the density using `sampleVolumeFast(pos)` [S1, unofficial].
        e.  **CTF Evaluation**:
            Evaluate color and alpha using `evaluateCTF(density)` [S1, unofficial].
        f.  **Alpha Threshold**:
            If `sampleAlpha <= 0.001`, skip lighting and accumulation, increment $t$ and `stepCount`, then `continue` [S1, unofficial].
        g.  **Alpha Normalization**:
            Compute normalized alpha:
            $$ \alpha = sampleAlpha \times \frac{stepSize}{0.006} $$
            Clamp $\alpha$ to $[0, 1]$ [S1, unofficial].
        h.  **Hybrid Lighting**:
            *   If `toggleGradient` is enabled:
                *   Sample the pre-computed gradient from `gradientTexture` at `pos`.
                *   Compute the lighting intensity using Phong shading with the gradient direction [S1, unofficial].
            *   **Opacity Threshold Check**:
                *   Only apply the computed lighting if the opacity change (or gradient magnitude) exceeds a threshold (typically 0.15) [S1, unofficial]. This prevents shading on smooth vessels [S1, unofficial].
                *   If the threshold is not met, set lighting intensity to 1.0 (or ambient) [S1, unofficial].
        i.  **Accumulation**:
            *   Compute weight: $weight = \alpha \times (1.0 - accumulatedAlpha)$ [S1, unofficial].
            *   Update accumulated color:
                $$ accumulatedColor += ctfColor.rgb \times lighting \times weight $$
            *   Update accumulated opacity:
                $$ accumulatedAlpha += weight $$
            *   If `accumulatedAlpha >= 0.95`, break the loop [S1, unofficial].
        j.  **Step Increment**:
            Increment $t$ by `stepSize` [S1, unofficial].
6.  **Post-Processing**:
    *   If `accumulatedAlpha < 0.1`, add an ambient gray color to mask numerical errors:
        $$ accumulatedColor += vec3(0.02) \times (0.1 - accumulatedAlpha) $$
        [S1, unofficial]
7.  **Return**:
    *   Return `vec4(accumulatedColor, accumulatedAlpha)` [S1, unofficial].

### Helper Functions

**Volume Sampling** [S1, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_volume_fast(uvw, volume_texture, num_slices, sampler):
    """
    Samples a 2D texture array volume.
    uvw: vec3<f32> in texture space [0,1].
    """
    coords = clamp(uvw, vec3(0.0), vec3(1.0))
    z = coords.z * (num_slices - 1.0)
    z0 = floor(z)
    z1 = min(z0 + 1.0, num_slices - 1.0)
    z_frac = z - z0
    
    # Sample slice z0
    sample0 = texture_sample_level(volume_texture, sampler, coords.xy, int(z0), 0.0)
    # Sample slice z1
    sample1 = texture_sample_level(volume_texture, sampler, coords.xy, int(z1), 0.0)
    
    return mix(sample0.r, sample1.r, z_frac)
```

**CTF Evaluation** [S1, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_ctf(density, ctf_texture):
    """
    Evaluates the Color Transfer Function.
    density: f32, HU value.
    """
    min_hu = -1024.0
    max_hu = 3071.0
    normalized = clamp((density - min_hu) / (max_hu - min_hu), 0.0, 1.0)
    texel_index = int(normalized * 255.0)
    return texture_load(ctf_texture, texel_index, 0)
```

### Edge Cases and Pitfalls
*   **Numerical Errors**: Very transparent regions (`accumulatedAlpha < 0.1`) may show black dots due to `f16` numerical errors. This is mitigated by adding the subtle gray ambient color in the post-processing step [S1, unofficial].
*   **Lasso Masking**: Voxels inside the lasso contour (`maskValue == 0`) are skipped. Ensure the mask texture is correctly aligned with the volume texture coordinates [S1, unofficial].
*   **Gradient Threshold**: Lighting is only applied if the opacity change exceeds 0.15. This prevents shading artifacts on smooth structures like blood vessels [S1, unofficial].
*   **Precision**: The method uses `f32` for uniforms and textures, but numerical errors can occur in `f16` contexts. The ambient gray fix is specific to this precision issue [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \times M)$, where $N$ is the number of pixels and $M$ is the average number of samples per ray. The number of samples is determined by the ray length through the volume and the `stepSize`.
*   **Memory Complexity**: $O(V)$, where $V$ is the volume of the texture data, plus overhead for the gradient texture, CTF texture, and lasso mask.
*   **Performance**: The method is implemented in a fragment shader, allowing for parallel execution on the GPU. The use of pre-computed gradients avoids the need for on-the-fly gradient calculation, saving computation time. The lasso mask check adds a small overhead per sample but allows for efficient culling of irrelevant voxels.
*   **Strengths**:
    *   Stable lighting due to pre-computed gradients.
    *   Selective rendering via lasso masking.
    *   Reduced artifacts on smooth structures via opacity threshold lighting.
*   **Limitations**:
    *   Requires pre-computation of gradients.
    *   The lasso mask must be provided and aligned with the volume.
    *   Numerical errors in low-opacity regions require a specific fix.

## Background
This method is described in the unofficial source [S1, unofficial], which provides a WebGPU-based volume renderer implementation. It is designed for medical imaging visualization, specifically for CT data, as indicated by the use of Hounsfield Units (HU) in the CTF normalization. The technique combines several established volume rendering concepts (ray marching, front-to-back compositing, transfer functions) with specific optimizations for medical data (HU gradients, lasso masking).

## Sources
[S1] dongeronimo/webgpu_volume_renderer_demo, https://github.com/dongeronimo/webgpu_volume_renderer_demo, fetched 2026-10-29 (unofficial)
