# Volume Shading Modes


## Overview
Volume Shading Modes are a set of three shading strategies applied during volume ray marching to compute lighting and shadows for each sample along a ray [S1, unofficial]. They extend the Phong Shading Model for Volume Data by providing specific heuristics for handling illumination in participating media, distinguishing themselves through their approach to gradient estimation and shadow accumulation [S1, unofficial].

## How it works
The method operates within a ray-marching loop, transforming rays to object space, sampling the volume, computing gradients, and applying shading based on the selected mode.

**Inputs and Outputs**
*   **Input**: Ray origin and direction in world space, volume data, transformation matrices (world-to-object, object-to-world), shading parameters (`shadingScale`, `gradientStep`), and light parameters (ambient color, directional light color and direction).
*   **Output**: Shaded color and accumulated alpha for the ray segment.

**Data Structures**
*   `RadiancePayload`: Contains `rng` (pointer to RandomTEA), `alpha` (float), and `color` (vec3f). Packaged via `packPointer(&payload, u0, u1)` for OptiX trace [S1, unofficial].

**Numbered Steps**
1.  **Transform Ray**: Transform ray origin and direction to object space [S1, unofficial]:
    ```cpp
    const vec3f org = xfmPoint(wto, _org);
    const vec3f dir = xfmVector(wto, _dir);
    ```
2.  **Initialize Tracking (Single Shade Heuristic)**: If using `SINGLE_SHADE_HEURISTIC`, initialize tracking variables [S1, unofficial]:
    ```cpp
    vec3f highestContributionOrg = 0.f;
    vec3f highestContributionColor = 0.f;
    float highestContributionAlpha = 0.f;
    ```
3.  **March Ray**: Iterate along the ray using `raymarching_iterator(self, org, dir, tMin, tMax, self.step, [&](const vec2f& t) { ... })` [S1, unofficial].
4.  **Sample Position**: Compute sample position with jitter [S1, unofficial]:
    ```cpp
    const auto p = org + lerp(jitter, t.x, t.y) * dir;
    ```
5.  **Compute Gradient**: Compute gradient in object space. Negate it because opacity correlates to the scalar field, and the gradient points toward increasing values (outside), requiring a flip for correct lighting [S1, unofficial]:
    ```cpp
    const vec3f No = -sampleGradient(self.volume, p, sampleValue, gradientStep);
    ```
6.  **Transform Normal**: Transform gradient to world space [S1, unofficial]:
    ```cpp
    const vec3f Nw = xfmNormal(otw, No);
    ```
7.  **Apply Shading**:
    *   **GRADIENT_SHADING**: Compute shaded color and blend [S1, unofficial]:
        ```cpp
        const vec3f shadingColor = shade_scivis_light(_dir, Nw, sampleColor, params.mat_gradient_shading, params.light_ambient, params.light_directional_rgb, params.light_directional_dir);
        sampleColor = lerp(shadingScale, sampleColor, shadingColor);
        ```
    *   **SINGLE_SHADE_HEURISTIC**: Track the point with the highest contribution [S1, unofficial]:
        ```cpp
        if (highestContributionAlpha < (1.f - payload.alpha) * sampleAlpha) {
            highestContributionOrg = p;
            highestContributionColor = sampleColor;
            highestContributionAlpha = (1.f - payload.alpha) * sampleAlpha;
        }
        ```
8.  **Continue Marching**: Return `payload.alpha < nearly_one` to continue marching while alpha is below threshold [S1, unofficial].

**Parameters**
*   `shadingScale`: Controls the interpolation weight between base color and shaded color in `lerp(shadingScale, base, shaded)` [S1, unofficial].
*   `gradientStep`: Step size used for finite-difference gradient estimation in `sampleGradient` [S1, unofficial].

**Pitfalls**
*   **Gradient Direction**: The gradient direction must be negated (`-sampleGradient`) because the implementation assumes opacity directly correlates to the volume scalar field, meaning the gradient naturally points toward higher values (outside), requiring a flip for correct lighting [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] GitHub - VIDILabs/open-volume-renderer: Open source modular multi ..., https://github.com/VIDILabs/open-volume-renderer, fetched 2026-10-29 (unofficial)
