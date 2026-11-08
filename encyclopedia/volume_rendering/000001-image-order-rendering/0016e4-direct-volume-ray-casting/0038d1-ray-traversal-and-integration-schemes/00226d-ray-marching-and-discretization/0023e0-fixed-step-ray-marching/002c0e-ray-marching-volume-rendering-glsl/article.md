# Ray Marching Volume Rendering (GLSL)

Also called: Fragment Shader Ray Marching, GLSL Ray Marching Volume Rendering, GPU Ray Marching, GPU-Based Ray Marching, Ray Marching on the GPU, Ray Marching with Trilinear Interpolation, vol-fs.glsl raymarch, vol-fs2.glsl raymarch

## Overview
Ray Marching Volume Rendering is a direct volume rendering technique that discretizes viewing rays into fixed steps to sample a 3D texture, accumulating color and opacity via front-to-back compositing within a fragment shader [S1, S2, S3, S4, unofficial]. It solves the problem of real-time volumetric visualization by performing the entire rendering integral in a single pass on the GPU, avoiding the memory bandwidth bottlenecks of slice-based methods like Shear-Warp [S4, unofficial]. Unlike its siblings, it processes rays independently, enabling optimizations such as early ray termination and adaptive sampling, and is generally faster than Shear-Warp for equivalent visual quality [S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A 3D volume texture (density/scalar field), a transfer function (optional, for color mapping), camera position and orientation matrices, and a bounding box defining the volume extent (typically a unit cube centered at origin or [0,1]^3) [S1, S2, S3, S4, unofficial].
*   **Outputs:** A final pixel color (vec4) with accumulated alpha [S1, S3, S4, unofficial].

**Data Structures:**
*   **Volume Texture:** Stored as a 3D texture (`sampler3D`) or a 2D texture atlas where slices are packed [S2, S4, unofficial]. If using a 2D atlas, the z-coordinate is mapped to slice index [S2, unofficial].
*   **Uniforms:**
    *   `u_volume_size` (ivec3): Dimensions of the volume [S1, unofficial].
    *   `u_matrix_translation` (mat4): World-to-local transform [S1, unofficial].
    *   `u_matrix_camera_translation` (mat4): Camera-to-world transform [S1, unofficial].
    *   `uTexDim` (vec3): Texture dimensions (width, height, depth) [S2, unofficial].
    *   `uTMK` / `uTMK2` (float): Transmittance multiplier constant for exponential absorption [S2, unofficial].
    *   `u_enable_max_value_sampling` (int): 1 for max-value sampling, 0 for standard compositing [S1, unofficial].
    *   `MAX_STEPS` (int): Maximum iteration count (e.g., 128, 200, 256) [S2, S3, unofficial].
    *   `STEP_SIZE` (float): Fixed step distance (e.g., 0.01) [S3, unofficial].
    *   `TM_MIN` (float): Minimum transmittance threshold for early termination (e.g., 0.25 or 0.0) [S2, unofficial].

**Numbered Steps:**

1.  **Preprocessing:** Read volume data from disk and estimate normals if required for shading [S2, S4, unofficial].
2.  **Ray Setup:**
    *   Compute ray origin in local volume space: `rayOrigin = (invModel * vec4(cameraPos, 1.0)).xyz` [S1, S2, S4, unofficial].
    *   Compute ray direction in camera space: `rayDir_cam = normalize(cam_rel_pos.xyz)` [S1, unofficial].
    *   Transform ray direction to local volume space: `rayDir_local = normalize((camToLocal * vec4(rayDir_cam, 0.0)).xyz)` [S1, unofficial].
    *   Alternatively, compute direction as `normalize(vPosition - rayOrigin)` [S2, unofficial].
3.  **Ray-Box Intersection:**
    *   Define bounding box: `boxMin = vec3(-0.5)`, `boxMax = vec3(0.5)` (or [0,1]) [S1, S3, unofficial].
    *   Compute intersection interval `[tMin, tMax]` using the slab method:
        $$ invDir = 1.0 / (rayDir + \epsilon) $$
        $$ t0s = (boxMin - rayOrigin) * invDir $$
        $$ t1s = (boxMax - rayOrigin) * invDir $$
        $$ tSmaller = min(t0s, t1s) $$
        $$ tLarger = max(t0s, t1s) $$
        $$ tMin = max(max(tSmaller.x, tSmaller.y), max(tSmaller.z, 0.0)) $$
        $$ tMax = min(min(tLarger.x, tLarger.y), tLarger.z) $$
    *   If `tMin >= tMax`, discard the fragment (ray misses volume) [S3, unofficial].
    *   Clamp `tMin` to 0.0 [S1, unofficial].
4.  **Step Size Calculation:**
    *   **Fixed Step:** Use a constant `STEP_SIZE` (e.g., 0.01) or derive from `MAX_STEPS` and diagonal length: `gStepSize = sqrt(3.0) / float(MAX_STEPS)` [S1, S2, S3, unofficial].
    *   **Voxel-Aligned Step:** Calculate `dt` to ensure at least one sample per voxel dimension:
        $$ size = u\_volume\_size * abs(rayDir\_local) $$
        $$ dt\_vec = 1.0 / size $$
        $$ dt = min(dt\_vec.x, min(dt\_vec.y, dt\_vec.z)) $$
        [S1, unofficial].
5.  **Ray Marching Loop:**
    *   Initialize `t = tMin`, `accumulatedColor = vec4(0.0)`, `tm = 1.0` (transmittance) [S1, S2, S3, unofficial].
    *   Loop for `i` from 0 to `MAX_STEPS` while `t < tMax`:
        a.  **Sample Position:** `pos = rayOrigin + t * rayDir_local` [S1, S3, unofficial].
        b.  **Texture Sampling:**
            *   Map `pos` to texture coordinates (0.0 to 1.0): `samplePos = pos + vec3(0.5)` [S1, unofficial].
            *   If using 2D atlas: Calculate `zSlice = (1.0 - pos.y) * (uTexDim.z - 1.0)`, sample two slices with `texture2D`, and lerp using `fract(zSlice)` [S2, unofficial].
            *   Read density: `density = texture(volumeTexture, samplePos).r` [S3, unofficial].
        c.  **Color/Alpha Mapping:**
            *   Map density to color: `voxel_color = texture(transfer_fn, vec2(density, 0.5))` [S1, unofficial].
            *   Calculate alpha contribution: `a = density * voxel_color.a` [S1, unofficial].
        d.  **Compositing:**
            *   **Standard Front-to-Back:**
                $$ accumulatedColor.rgb += (1.0 - accumulatedColor.a) * a * voxel\_color.rgb $$
                $$ accumulatedColor.a += (1.0 - accumulatedColor.a) * a $$
                [S1, S3, unofficial].
            *   **Max-Value Sampling:** If `a > accumulatedColor.a`, update `accumulatedColor.rgb = voxel_color.rgb * a` and `accumulatedColor.a = max(a, accumulatedColor.a)` [S1, unofficial].
            *   **Lighting (Optional):** Compute light direction `ld`, light transmittance `ltm`, and add: `col += (1.0 - dtm) * uColor2 * uLightC[k] * tm * ltm` [S2, unofficial].
        e.  **Update Transmittance:** `tm *= exp(-uTMK * density * step)` [S2, unofficial].
        f.  **Advance Ray:** `t += step` [S1, S3, unofficial].
        g.  **Early Termination:**
            *   If `accumulatedColor.a >= 0.95`, break loop [S1, S3, unofficial].
            *   If `tm < TM_MIN`, break loop [S2, unofficial].
            *   If `pos` is outside bounds, break loop [S2, unofficial].
6.  **Final Output:**
    *   Compute final alpha: `alpha = 1.0 - tm` [S2, unofficial].
    *   Return `vec4(accumulatedColor.rgb / alpha, alpha)` [S2, unofficial].
    *   Discard if `FragColor.a <= 0.001` [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```glsl
// Inputs: volumeTexture (sampler3D), transfer_fn (sampler2D), uniforms
// Outputs: FragColor (vec4)

void main() {
    // 1. Ray Setup
    vec3 rayOrigin = (invModel * vec4(cameraPos, 1.0)).xyz;
    vec3 rayDir_cam = normalize(cam_rel_pos.xyz);
    vec3 rayDir_local = normalize((camToLocal * vec4(rayDir_cam, 0.0)).xyz);

    // 2. Ray-Box Intersection
    vec3 invDir = 1.0 / (rayDir_local + 0.0000001); // Epsilon for division by zero
    vec3 t0s = (boxMin - rayOrigin) * invDir;
    vec3 t1s = (boxMax - rayOrigin) * invDir;
    vec3 tSmaller = min(t0s, t1s);
    vec3 tLarger = max(t0s, t1s);
    float tMin = max(max(tSmaller.x, tSmaller.y), max(tSmaller.z, 0.0));
    float tMax = min(min(tLarger.x, tLarger.y), tLarger.z);

    if (tMin >= tMax) {
        discard;
    }
    tMin = max(tMin, 0.0);
    float travelDistance = tMax - tMin;
    
    // Step size: Fixed or voxel-aligned
    float step = STEP_SIZE; 
    // Alternative: float step = travelDistance / float(MAX_STEPS);

    // 3. Initialization
    vec4 accumulatedColor = vec4(0.0);
    float tm = 1.0;
    float t = tMin;

    // 4. Ray Marching Loop
    for (int i = 0; i < MAX_STEPS && t < tMax; ++i) {
        vec3 pos = rayOrigin + t * rayDir_local;
        vec3 samplePos = pos + vec3(0.5); // Offset for texture coords

        // Sample volume
        float density = texture(volumeTexture, samplePos).r;
        
        // Map to color
        vec4 voxel_color = texture(transfer_fn, vec2(density, 0.5));
        float a = density * voxel_color.a;

        // Compositing
        if (u_enable_max_value_sampling == 1) {
            if (a > accumulatedColor.a) {
                accumulatedColor.rgb = voxel_color.rgb * a;
                accumulatedColor.a = max(a, accumulatedColor.a);
            }
        } else {
            accumulatedColor.rgb += (1.0 - accumulatedColor.a) * a * voxel_color.rgb;
            accumulatedColor.a += (1.0 - accumulatedColor.a) * a;
        }

        // Update transmittance
        tm *= exp(-uTMK * density * step);

        // Advance
        t += step;

        // Early Termination
        if (accumulatedColor.a >= 0.95) break;
        if (tm < TM_MIN) break;
    }

    // 5. Final Output
    float alpha = 1.0 - tm;
    if (alpha <= 0.001) {
        discard;
    }
    FragColor = vec4(accumulatedColor.rgb / alpha, alpha);
}
```

**Parameters:**
*   `MAX_STEPS`: 128 (vol-fs.glsl), 200 (vol-fs2.glsl), 256 [S2, S3, unofficial].
*   `STEP_SIZE`: 0.01 [S3, unofficial].
*   `TM_MIN`: 0.25 (vol-fs.glsl), 0.0 (vol-fs2.glsl) [S2, unofficial].
*   Early Ray Termination Threshold: 0.95 [S1, S3, unofficial].
*   Sampling Rate (s): In Shear-Warp context, equivalent to virtual slices; for Ray Marching, this corresponds to `MAX_STEPS` or inverse step size [S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** Ray direction components can be zero; handle with epsilon in `invDir` [S1, unofficial].
*   **Aliasing:** Uniform fixed-step sampling causes aliasing artifacts if the sampling rate is too low [S4, unofficial].
*   **Visible Slices:** Low sampling rates make individual slices visible; solved by increasing steps or using interpolation [S4, unofficial].
*   **Texture Coordinate Layout:** Non-standard layouts (e.g., z-up in tex coords) require specific handling in `sampleVolTex` [S2, unofficial].

## Subtopics

### Ray Marching Volume Rendering (1735-ray-marching-volume-rendering.md)
General fixed-step ray marching. [Link](1735-ray-marching-volume-rendering.md)

### Ray Evaluation Stepping DVR (3461-ray-evaluation-stepping-dvr.md)
Adaptive stepping based on density gradients. [Link](3461-ray-evaluation-stepping-dvr.md)

### Density Marching (3462-density-marching.md)
Marching based on density thresholds rather than fixed steps. [Link](3462-density-marching.md)
*   **Discrete Volume Rendering Integral (VRI) (3552-discrete-volume-rendering-integral-vri.md):** Analytical integration over steps. [Link](3552-discrete-volume-rendering-integral-vri.md)
*   **Volumetric Light Shafts (URP Raymarching) (3873-volumetric-light-shafts-urp-raymarching.md):** Ray marching for light scattering in URP. [Link](3873-volumetric-light-shafts-urp-raymarching.md)
*   **Cloud Volume Rendering (4877-cloud-volume-rendering.md):** Specific optimizations for cloud volumes. [Link](4877-cloud-volume-rendering.md)

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the number of steps per ray [S4, unofficial].
*   **Memory Complexity:** $O(V)$ for the 3D volume texture, where $V$ is the number of voxels [S4, unofficial].
*   **Measured Performance:** On a 3.8GHz Pentium 4 with GeForce FX7950 GX2, GPU Ray Marching achieved 44 fps for a 512x512x73 Knee model at 500 steps, compared to 134 fps for Shear-Warp at sampling rate 1, but Ray Marching at 400 steps (48 fps) was faster than Shear-Warp at sampling rate 10 (10 fps) [S4, unofficial].
*   **Scaling:** Frame rate drops linearly with increased steps or sampling rate [S4, unofficial].
*   **Strengths:** Single-pass rendering, independent ray processing allows early termination and empty space skipping, higher quality than low-rate Shear-Warp [S4, unofficial].
*   **Limitations:** Aliasing artifacts at low sampling rates, performance degrades with high step counts, fixed step size may oversample empty space or undersample dense regions [S4, unofficial].
*   **When to Use:** Real-time visualization on standard PC hardware where simplicity and single-pass efficiency are prioritized over adaptive sampling [S4, unofficial].
*   **When Not to Use:** When high-quality anti-aliasing is required without significant performance cost, or when volumes have large empty regions where empty space skipping is critical (though early termination helps) [S4, unofficial].

## Background
Originated in the context of GPU-based volume rendering for medical imagery, specifically as a comparison to the Shear-Warp algorithm [S4, unofficial]. The method leverages the fragment shader's ability to perform parallel ray traversal, moving computational load from CPU to GPU [S4, unofficial]. It was implemented to address the limitations of slice-based methods, such as visible artifacts at low sampling rates and lower frame rates for high-quality rendering [S4, unofficial].

## Sources
[S1] jaanonim/volume-rendering, https://github.com/jaanonim/volume-rendering, fetched 2026-10-26 (unofficial)
[S2] rbaravalle/webgl-volumetric, https://github.com/rbaravalle/webgl-volumetric, fetched 2026-10-27 (unofficial)
[S3] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
[S4] GPU-Based Volume Rendering for Medical Imagery, https://scholarly.org/pdf/display/gpu-based-volume-rendering-for-medical-imagery, fetched 2026-10-28 (unofficial)
