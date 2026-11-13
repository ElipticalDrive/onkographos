# Cone Occlusion based on Gaussian Integrals

Also called: Cone Occlusion, Cone Shadow, RC1PConeTracingDirOcclusionShading

## Overview
Cone Occlusion based on Gaussian Integrals is a volume rendering shading technique that approximates light occlusion and shadowing by tracing cones through a voxel volume and integrating extinction coefficients using Gaussian distributions [S1, unofficial]. It supports two distinct modes: Cone Occlusion for self-shadowing (ambient occlusion) and Cone Shadow for external light source shadows [S1, unofficial]. It is distinguished from standard ray casting by replacing discrete step sampling with continuous Gaussian integration over conical volumes, reducing aliasing and providing smoother shading results [S1, unofficial].

## How it works
The method operates in a single-pass compute shader environment, utilizing pre-computed volume data and specific shader uniforms to calculate visibility.

### Inputs and Outputs
*   **Inputs:**
    *   Extinction coefficient volume texture (3D texture).
    *   Camera matrices: Eye, View, Projection.
    *   Light source positions (for Cone Shadow mode).
    *   Blinn-Phong coefficients ($K_{ambient}, K_{diffuse}, K_{specular}, N_{shininess}$).
    *   Cone tracing parameters (step size, angles, distances).
*   **Outputs:**
    *   Output image texture (screen-space shading result).

### Data Structures
The implementation relies on a `ConeGaussianSampler` structure to manage cone parameters. Key fields include:
*   `cone_half_angle`: Aperture half-angle in degrees.
*   `initial_step`: Initial gap step for cone sampling.
*   `d_sigma`: Cone sample separation multiplier.
*   `r_sigma`: Minimum circular cone section radius.
*   `covered_distance`: Maximum distance the cone covers.
*   `max_gaussian_packing`: Enum indicating the number of Gaussian sections packed into the cone (e.g., `_1`, `_3`).

### Parameters and Initialization
1.  **Step Size Calculation:** The integration step size $m\_u\_step\_size$ is initialized based on the volume scale $sv = (sv.x, sv.y, sv.z)$:
    $$ m\_u\_step\_size = \frac{0.5}{\sqrt{3.0}} \sqrt{sv.x^2 + sv.y^2 + sv.z^2} $$
    [S1, unofficial]
2.  **Cone Occlusion Mode Parameters:**
    *   `cone_half_angle`: Default 20.0 degrees.
    *   `initial_step`: Default $3.0 \times \sigma_0$, where $\sigma_0$ is the base level Gaussian sigma from the extinction volume generation.
    *   `covered_distance`: Set to $0.50 \times \text{volume diagonal}$.
    *   `max_gaussian_packing`: `_3`.
3.  **Cone Shadow Mode Parameters:**
    *   `cone_half_angle`: Default 0.5 degrees.
    *   `covered_distance`: Set to $0.75 \times \text{volume diagonal}$.
    *   `max_gaussian_packing`: `_1`.
4.  **Light Cache:** A pre-computed light cache texture with resolution $32 \times 32 \times 32$ is used for shading coefficients [S1, unofficial].

### Algorithm Steps
1.  **Initialization:**
    *   Verify the extinction coefficient volume texture is not null. If null, initialization fails [S1, unofficial].
    *   Generate cone samples and estimate the initial integration step size using the formula above [S1, unofficial].
2.  **Uniform Setup:**
    *   Bind camera matrices (Eye, View, Projection), FOV tangent, and aspect ratio.
    *   Set occlusion/shadow flags, step size, and Blinn-Phong coefficients [S1, unofficial].
3.  **Compute Shader Dispatch:**
    *   Clear the output texture.
    *   Bind the rendering shader and output image texture.
    *   Dispatch compute shader groups based on screen resolution or multi-scaling mode [S1, unofficial].
4.  **Per-Pixel Shading (Conceptual):**
    *   For each pixel, determine the surface point $p$ and normal.
    *   Trace a cone from $p$ along the relevant direction (hemisphere normal for occlusion, light direction for shadow).
    *   Sample the extinction coefficient volume along the cone's path.
    *   Integrate the extinction using Gaussian weights defined by the cone's geometry and `max_gaussian_packing` setting.
    *   Compute the final color using the Blinn-Phong model modulated by the calculated visibility/occlusion factor [S1, unofficial].

### Edge Cases and Pitfalls
*   **Null Volume:** If the current volume texture is null, the initialization process fails and returns false [S1, unofficial].
*   **Precision:** The use of Gaussian integrals requires careful handling of floating-point precision during the integration steps, particularly when `d_sigma` and `r_sigma` are adjusted via UI [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Single-pass compute shader execution. Complexity depends on the number of Gaussian sections packed (`max_gaussian_packing`) and the volume resolution.
*   **Memory Complexity:** Requires storage for the extinction coefficient volume, the light cache texture ($32^3$), and output image.
*   **Performance:** The method is designed for real-time or near-real-time performance on GPU-accelerated systems [S1, unofficial].
*   **Strengths:** Provides smoother shading than discrete ray casting; supports both ambient occlusion and directional shadows in a unified framework.
*   **Limitations:** Relies on pre-computed volumes and light caches; accuracy depends on the Gaussian approximation parameters.
*   **When to use:** Suitable for interactive volume visualization where smooth shading and shadowing are desired without the cost of full path tracing.

## Background
The method is implemented in the `cppvolrend` project, a volume rendering application developed using C++ and GLSL [S1, unofficial]. It is based on the paper "Directional Ambient Occlusion and Shadows with Cone Tracing" by Schott et al. (2009) and related work on voxel cone tracing for volume visualization [S1, unofficial]. The implementation aims to replicate results from various volume rendering papers, including those on extinction-based shading and directional occlusion [S1, unofficial].

## Sources
[S1] bazze23/kthvis_volrend, https://github.com/bazze23/kthvis_volrend, fetched 2026-10-26 (unofficial)
