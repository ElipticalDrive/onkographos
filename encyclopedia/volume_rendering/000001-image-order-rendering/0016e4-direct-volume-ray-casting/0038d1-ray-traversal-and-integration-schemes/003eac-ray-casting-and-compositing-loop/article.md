# Ray Casting and Compositing Loop

Also called: Basic Ray Casting, Ray Casting Volume Rendering, Single pass GPU Ray casting, Standard Ray Casting Loop

## Overview
Ray Casting and Compositing Loop is the fundamental image-order rendering method for direct volume visualization, where scalar volume data is sampled along rays and mapped directly to color and opacity without intermediate surface extraction [S1, unofficial]. It differs from iso-surface methods by avoiding zero-crossing detection and normal computation, instead relying on front-to-back alpha compositing to accumulate optical properties along the ray path [S1, unofficial]. This method serves as the base engine for [Ray Traversal and Integration Schemes](8341-ray-traversal-and-integration-schemes.md), providing the core loop structure that acceleration techniques like [Wavelet-Assisted Volume Ray Casting](10893-wavelet-assisted-volume-ray-casting.md) optimize.

## How it works
The method executes a single-pass loop per pixel, typically implemented in a fragment shader on the GPU [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** Volume texture (3D texture sampler), camera position, ray direction, step size, maximum sample count.
*   **Output:** Final color and opacity (`FragColor`, a `vec4`) for the pixel.

**Data Structures:**
*   **Volume Data:** Stored in a 3D texture (`sampler3D`) allowing trilinear interpolation [S1, unofficial].
*   **Ray State:** Current position (`dataPosition`), accumulated color (`FragColor.rgb`), accumulated opacity (`FragColor.a`).

**Parameters:**
*   `MAX_SAMPLES`: Integer constant defining the maximum number of steps per ray. Typical value: 300 [S1, unofficial].
*   `u_stepSize`: Uniform vector defining the distance between samples [S1, unofficial].
*   `Early Ray Termination Threshold`: Opacity value at which the loop breaks. Typical value: 0.99 [S2, unofficial].

**Steps:**

1.  **Ray Generation:**
    Calculate the ray origin and direction. The origin `oTexCoords` is interpolated from 3D texture coordinates [S1, unofficial]. The geometry direction is computed by subtracting the camera position from the origin and normalizing:
    $$ \mathbf{d} = \text{normalize}(\mathbf{o} - \mathbf{cameraPosition}) $$
    [S1, unofficial]. The ray segment intersecting the volume is determined by intersecting with the volume's bounding box [S8, S16, unofficial].

2.  **Initialization:**
    Initialize the current position `dataPosition` to the ray origin [S1, unofficial]. Initialize accumulated color `FragColor.rgb` to $(0, 0, 0)$ and accumulated opacity `FragColor.a` to $0$ [S15, S46, unofficial].

3.  **Sampling Loop:**
    Iterate $i$ from $0$ to `MAX_SAMPLES` [S1, unofficial]:
    a.  **Boundary Check:** If the current position `dataPosition` is outside the volume bounds, break the loop. A common GPU implementation checks if the dot product of the signed distance vectors to the min and max bounds is less than 3.0:
        $$ \text{dot}(\text{sign}(\mathbf{p} - \mathbf{min}), \text{sign}(\mathbf{max} - \mathbf{p})) < 3.0 \implies \text{break} $$
        [S1, unofficial].
    b.  **Sample:** Retrieve the scalar value $V$ from the volume at `dataPosition` using trilinear interpolation [S9, S11, S47, unofficial].
    c.  **Classification:** Map $V$ to color $C_{sample}$ and opacity $\alpha_{sample}$ using a transfer function. The opacity is often corrected for the step size $\Delta t$ using the Beer-Lambert law approximation:
        $$ \alpha_{sample} = 1 - e^{-\sigma \Delta t} $$
        where $\sigma$ is the extinction coefficient derived from $V$ [S24, S30, unofficial].
    d.  **Compositing:** Update the accumulated color and opacity using the front-to-back compositing equations [S1, unofficial][S2, S13, unofficial][S61]:
        $$ \alpha_{out} = \alpha_{in} + (1 - \alpha_{in}) \cdot \alpha_{sample} $$
        $$ C_{out} = C_{in} + (1 - \alpha_{in}) \cdot C_{sample} \cdot \alpha_{sample} $$
        [S52, S60, unofficial]. In the specific implementation of [S1, unofficial], this is calculated as:
        ```python
        prev_alpha = alpha_sample * (1.0 - FragColor.a)
        FragColor.rgb = prev_alpha * C_sample + FragColor.rgb
        FragColor.a += prev_alpha
        ```
    e.  **Early Ray Termination:** If `FragColor.a` exceeds the threshold (e.g., 0.99), break the loop [S2, unofficial].
    f.  **Advance:** Update `dataPosition` by adding the step vector:
        $$ \mathbf{p}_{new} = \mathbf{p}_{old} + \mathbf{d} \cdot \Delta t $$
        [S1, unofficial].

**Pitfalls:**
*   **Boundary Intersection:** Accurate intersection with the volume boundary is critical, especially for methods like MLS ray casting where the approximation is ill-defined away from data points [S2, unofficial].
*   **Precision:** Floating-point precision errors in position accumulation can lead to gaps or overlaps in sampling over long distances [G].
*   **Step Size:** Too large a step size causes aliasing and loss of detail; too small increases computational cost without visual benefit [S1, unofficial].

## Subtopics

### [Wavelet-Assisted Volume Ray Casting](10893-wavelet-assisted-volume-ray-casting.md)
An acceleration technique that uses wavelet coefficients to detect local frequency content and determine an adaptive sampling distance along each ray. It solves the problem of excessive computational cost in standard ray casting by avoiding on-the-fly reconstruction from the wavelet domain; instead, it pre-computes an "index volume" that guides the traversal of the original spatial volume. Unlike Westermann's method, which traverses wavelet coefficients directly, this approach uses a flat index volume to skip samples, combining spatial domain rendering with wavelet-based frequency analysis.

### [Volumetric Shading](14296-volumetric-shading.md)
A technique for capturing shadow information and global shadow effects in translucent media by branching a secondary ray towards the light source at each primary ray-cast sample location. It computes transmittance at runtime to reduce memory occupancy compared to shadow maps. Unlike [Wavelet-Assisted Volume Ray Casting](10893-wavelet-assisted-volume-ray-casting.md), which optimizes traversal, volumetric shading focuses on the lighting model by explicitly integrating occlusion along the path to the light, providing perceptual cues for spatial relations and object size.

### Multi-Cast / Super-Sampling Ray Casting
Instead of sampling one ray per pixel, this variant samples 4 rays per pixel to improve image quality by reducing aliasing [S3, unofficial]. It produces better image quality but takes four times longer to render [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of samples per ray and $P$ is the number of pixels. The complexity is linear with respect to the number of samples and pixels [S3, unofficial].
*   **Memory Complexity:** $O(V)$, where $V$ is the size of the volume data, stored in a 3D texture [S1, unofficial].
*   **Performance:** The method is efficient for interactive rendering on modern GPUs due to its single-pass nature and parallelizability [S1, unofficial]. However, it can be computationally expensive for high-resolution volumes or large step counts [S3, unofficial].
*   **Strengths:** Simple to implement, produces high-quality results with proper transfer functions, supports early ray termination for optimization [S1, S2, unofficial].
*   **Limitations:** Sensitive to step size, can suffer from aliasing if not super-sampled, performance degrades with large volumes or complex transfer functions [S3, unofficial].
*   **When to use:** Ideal for direct volume visualization where surface extraction is not required, and interactive performance is needed [S1, unofficial].
*   **When not to use:** Not suitable for scenes requiring complex lighting effects like shadows without additional techniques (e.g., Volumetric Shading) [S1, unofficial].

## Background
The method originated as a fundamental technique in direct volume rendering, leveraging the efficiency of GPU fragment shaders for single-pass rendering [S1, unofficial]. It is widely used in medical imaging and scientific visualization for its simplicity and effectiveness [S1, unofficial]. The front-to-back compositing equation is a standard technique in computer graphics for alpha blending [S1, unofficial][S2, S13, unofficial][S61].

## Sources
[S1] Volumetric Render Engine - GitHub, https://github.com/mikejernil/volumetric-render-engine, fetched 2026-10-27 (unofficial)
[S2] Volume MLS Ray Casting, https://miriah.github.io/publications/volume_mls.pdf, fetched 2026-10-29 (unofficial)
[S3] Introduction to volume rendering, https://slidetodoc.com/introduction-to-volume-rendering-presented-by-zvi-devir/, fetched 2026-10-29 (unofficial)
