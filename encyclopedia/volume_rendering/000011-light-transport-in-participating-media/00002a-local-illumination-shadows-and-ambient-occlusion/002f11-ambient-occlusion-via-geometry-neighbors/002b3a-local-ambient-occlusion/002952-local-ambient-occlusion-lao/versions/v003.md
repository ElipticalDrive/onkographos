# Local Ambient Occlusion (LAO)

Also called: LAO Renderer, Local Ambient Occlusion (LAO) in DVR, Volumetric Ambient Occlusion

## Overview
Local Ambient Occlusion (LAO) is a technique for estimating local occlusion in volume rendering by sampling the volume along random directions around the light vector, weighted by distance, to simulate ambient occlusion effects [S1, unofficial]. It differs from global shadowing methods by approximating indirect illumination through incremental, per-pixel accumulation of visibility samples, allowing for dynamic interaction with the volume's internal density structure without precomputed global buffers [S2, unofficial].

## How it works
The method operates in two main phases: an incremental LAO map computation and a final ray-casting pass that utilizes the precomputed occlusion data.

### Inputs and Outputs
*   **Inputs:** Volume data (scalar field), light direction $\mathbf{l}$, camera parameters, and LAO configuration parameters (see Parameters).
*   **Outputs:** A final rendered image with ambient occlusion applied, and optionally a LAO map stored in a Frame Buffer Object (FBO) [S2, unofficial].

### Data Structures
1.  **LAO Map:** Stored in a Frame Buffer Object (FBO). The FBO planes correspond to screen coordinates $(x, y)$ and store intensity/light computation results [S2, unofficial].
2.  **Accumulator:** A temporary buffer (`accumuLAOContribution`) used to accumulate LAO contributions during the sampling loop [S1, unofficial].
3.  **Packed Coordinates:** For multiresolution volumes, indices map between Volume-to-Pixel ($V \to P$) and Pixel-to-Volume ($P \to V$) to efficiently access data in packed textures [S2, unofficial].

### Parameters
*   `uLocalAmbientOcclusion`: Boolean flag to enable/disable LAO calculation. Default: `true` [S1, unofficial].
*   `uNumLAOSamples`: Number of LAO samples per pixel (directions). Default: `1` [S1, unofficial].
*   `uLAOStepSize`: Step size for LAO ray marching. Default: `0.05` [S1, unofficial].
*   `uLightRadious`: Radius of the light source used for LAO sampling. Default: `0.19` [S1, unofficial].
*   `uLightCoeficient`: Coefficient used to normalize LAO accumulation. Default: `1.0` [S1, unofficial].
*   `uLAOWeight`: Weight applied to the LAO contribution when mixing into the final color. Default: `0.69` [S1, unofficial].
*   `K`: Number of directions for LAO computation (e.g., 1 or 8) [S2, unofficial].

### Algorithm Steps

#### Phase 1: Incremental LAO Map Computation
This phase updates the LAO map incrementally, typically adding one ray direction per frame to converge on the final occlusion value [S2, unofficial].

1.  **Initialize Accumulator:** Reset the LAO accumulator for the current pixel to zero.
    ```python
    accumuLAOContribution = vec4(0.0)
    ```
2.  **Loop over Directions:** Iterate $K$ times (or `uNumLAOSamples` times) to sample different directions [S1, unofficial][S2, unofficial].
    *   **Generate Random Direction:** Create a random direction vector.
        ```python
        # Generate random vector in [-1, 1]^3 and normalize
        random_dir_raw = normalize(vec3(rand(), rand(), rand()) * 2.0 - 1.0)
        # Apply additional random scaling if required by specific implementation
        randomDirection = random_dir_raw * rand()
        ```
    *   **Ray Marching:** Loop through the sampling distance $t$ from `0.001` to `1.0` with step `uLAOStepSize` [S1, unofficial].
        *   **Compute Half Vector:** Calculate the sampling position relative to the light and random direction.
            ```python
            # vLight is the light direction vector
            # position is the current surface/volume point
            laoHalfVector = normalize(vLight + randomDirection * mix(0.0, uLightRadious, t) - position)
            ```
        *   **Sample Volume:** Sample the volume density at `position + laoHalfVector * t` [S1, unofficial].
        *   **Accumulate Contribution:** Add the sample value to the accumulator, weighted by distance. The weight is $(1.0 - t)^2$ [S1, unofficial].
            ```python
            laoSample = sample_volume(position + laoHalfVector * t)
            accumuLAOContribution += laoSample * pow(1.0 - t, 2.0)
            ```
    *   **Normalize Accumulator:** After the inner loop, divide the accumulator by `uLightCoeficient` and clamp the result to $[0, 1]$ [S1, unofficial].
        ```python
        LAOContribution = clamp(accumuLAOContribution / uLightCoeficient, 0.0, 1.0)
        ```
3.  **Finalize Direction Average:** After all $K$ directions are processed, divide the total `LAOContribution` by `uNumLAOSamples` [S1, unofficial].
    ```python
    LAOContribution = LAOContribution / uNumLAOSamples
    ```
4.  **Store Result:** Write `LAOContribution` to the LAO map in the FBO [S2, unofficial].

#### Phase 2: Final Ray Casting
The final rendering pass uses the LAO map to modulate the volume's appearance.

1.  **Map Coordinates:** Convert screen coordinates to volume coordinates using $P \to V$ and $V \to P$ indices [S2, unofficial].
2.  **Ray Cast:** Perform standard volume ray casting.
3.  **Apply LAO:** Retrieve the LAO value from the FBO for the current pixel.
4.  **Shading:** Compute the final color using Transfer Functions for Color ($TF_c$) and Emission ($TF_E$). The ambient occlusion factor modulates the diffuse and emissive components [S2, unofficial].
    *   If emission is enabled, add the emissive color $C_E$ to the integral [S2, unofficial].

### Edge Cases and Pitfalls
*   **Empty Space:** In multiresolution volumes, empty space skipping is used. If a volume block is smaller than $4^3$ voxels, it may be skipped to reduce computation [S2, unofficial].
*   **Low Resolution:** For low-resolution blocks, the sampling distance should be increased to maintain visual consistency [S2, unofficial].
*   **Precision:** The incremental update requires multiple frames to converge. Stopping early results in noisy or incomplete occlusion [S2, unofficial].

## Subtopics

### In-scattering / Emittance Integral
A discretized compositing scheme for computing in-scattered light using an emittance integral, implemented via progressive refinement with one direction per pass [S2, unofficial]. It is similar to LAO but applied to in-scattered/emissive light contributions rather than just shadowing [S2, unofficial].
*   **Steps:** Discretize the emittance integral into a compositing scheme and apply progressive refinement, computing one light direction per rendering pass [S2, unofficial].
*   **Parameters:** `Ibias` (Minimum light value clamped for a volume sample to prevent dark artifacts). Typical value: $0.0$ to $0.2$ [S2, unofficial].
*   **Performance:** Translucency effect parameters include Ray Count ($R_\Omega$) of 16 or 48. Higher $R_\Omega$ and `Ibias` increase brightness and reduce noise [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The method is incremental. Updating one ray in the LAO map takes approximately 51 ms on an NVIDIA GF8800 Ultra with 768 MB graphics texture memory, resulting in 27 FPS for a single direction [S2, unofficial]. Using 8 directions would proportionally increase the time per frame if updated simultaneously, but incremental updates allow for smoother convergence over time.
*   **Memory Complexity:** Requires an FBO for the LAO map. For multiresolution volumes, memory usage is optimized by storing packed textures with varying resolutions (High, Middle, Low) and skipping empty blocks [S2, unofficial].
*   **Strengths:** Allows for dynamic lighting and occlusion in real-time volume rendering without expensive global precomputation. Multiresolution support provides super-linear performance increases for large volumes (e.g., $512^3$ voxels) by reducing the number of voxels to compute [S2, unofficial].
*   **Limitations:** Convergence is slow due to incremental updates. Noise may be visible in early frames. Performance is sensitive to volume density and resolution [S2, unofficial].
*   **When to use:** When real-time or near-real-time ambient occlusion is needed for volumes with complex internal structures, and global shadowing is too expensive.
*   **When not to use:** When high-quality, noise-free global illumination is required immediately, or when the volume is static and precomputed lighting is feasible.

## Background
Local Ambient Occlusion for volume rendering was developed by Patric Ljung at Siemens Corporate Research, Princeton, USA, as part of his work on "Advanced Illumination Techniques for GPU-Based Volume Raycasting" [S2, unofficial]. The technique was presented in 2008 [S2, unofficial]. It builds upon earlier work on multiresolution volume rendering and local shading models, aiming to provide a practical solution for ambient occlusion in interactive volume visualization [S2, unofficial].

## Sources
[S1] terier/vpt, https://github.com/terier/vpt, fetched 2026-10-26 (unofficial)
[S2] GPU-Based Volume Ray-Casting with Advanced Illumination, https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2008/advanced_illumination.pdf, fetched 2026-10-29 (unofficial)
