# Horizon Zero Dawn Volumetric Cloud System

Also called: Horizon Clouds, Procedural Volumetric Clouds for Horizon Zero Dawn

## Overview
The Horizon Zero Dawn Volumetric Cloud System is a real-time procedural method for generating realistic, evolving cloudscapes by combining Perlin noise with Worley noise offsets, stored in compressed 3D and 2D textures [S1, unofficial]. It solves the limitations of previous asset-based approaches (billboards, sky domes) which failed to support inter-cloud shadowing, temporal evolution, and overhead cloud movement while incurring high memory usage and overdraw [S1, unofficial]. Unlike standard fractal Brownian motion (fBm) which lacks large governing shapes, this system uses "Perlin-Worley" noise to create packed, billowy, cauliflower-like structures that imply motion and evolution [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** 3D sample position $\mathbf{P}$ (world or camera space), altitude/height signal, and coverage signal.
*   **Output:** Cloud density and shape data for volume rendering.

**Data Structures**
The system relies on three pre-computed noise textures to minimize runtime computation and texture reads [S1, unofficial]:

1.  **Noise Texture 1 (Base Shape):**
    *   Resolution: $128^3$
    *   Channels: 4
    *   Layout:
        *   Channel 0: Perlin-Worley noise (base shape).
        *   Channels 1–3: Layered Worley noise at increasing frequencies.
2.  **Noise Texture 2 (Detail):**
    *   Resolution: $32^3$
    *   Channels: 3
    *   Layout: Worley noise at increasing frequencies to add detail to the base shape.
3.  **Noise Texture 3 (Turbulence):**
    *   Resolution: $128^2$
    *   Channels: 3
    *   Layout: Curl noise (non-divergent) used to distort shapes and fake fluid motion.

**Parameters**
*   **Height Signal:** A mathematical preset representing major low-altitude cloud types. It is used to blend between cloud types at the sample position [S1, unofficial].
*   **Coverage Signal:** A scalar value between 0 and 1 indicating the desired cloud coverage at the sample position [S1, unofficial].

**Algorithm Steps**
1.  **Base Shape Generation:** Sample `3DTexture1` at position $\mathbf{P}$. The result is multiplied by the `Height Signal` to build the low-resolution base cloud shape. This combines the connectedness of Perlin noise with the dilated, billowy shapes of Worley noise [S1, unofficial].
2.  **Detail and Turbulence:** Sample `3DTexture2` for high-frequency Worley detail and `2DTexture3` for curl noise. The curl noise is used to distort the cloud shape, simulating turbulence and fluid motion [S1, unofficial].
3.  **Erosion and Coverage:** Erode the result from Step 1. Multiply the eroded shape by the `Coverage Signal` (0 to 1) [S1, unofficial].
4.  **Base Density Reduction:** Reduce density at the base of the cloud to create whispy bottoms. This ensures that density increases naturally with altitude, enhancing realism [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_horizon_cloud_density(P, height_signal, coverage_signal):
    # Step 1: Sample Base Shape from 3DTexture1 (128^3, 4 channels)
    # Channel 0 is Perlin-Worley, Channels 1-3 are layered Worley
    base_noise = sample_3d_texture(3DTexture1, P)
    
    # Multiply by height signal to blend cloud types based on altitude
    base_shape = base_noise * height_signal
    
    # Step 2: Sample Detail from 3DTexture2 (32^3, 3 channels)
    # Adds high-frequency Worley detail
    detail_noise = sample_3d_texture(3DTexture2, P)
    
    # Step 3: Sample Turbulence from 2DTexture3 (128^2, 3 channels)
    # Curl noise for non-divergent distortion
    turbulence = sample_2d_texture(2DTexture3, P.xy)
    
    # Apply turbulence to distort the shape
    distorted_shape = base_shape + turbulence
    
    # Step 4: Erode the result
    eroded_shape = erode(distorted_shape)
    
    # Step 5: Multiply by coverage signal
    covered_shape = eroded_shape * coverage_signal
    
    # Step 6: Reduce density at the base for whispy bottoms
    # This assumes 'P.y' or a derived altitude metric is available
    altitude_factor = compute_altitude_factor(P)
    final_density = covered_shape * altitude_factor
    
    return final_density
```

**Edge Cases and Pitfalls**
*   **Precision:** The system uses compressed textures to save memory, which may introduce quantization artifacts if resolution is too low. The chosen resolutions ($128^3$ and $32^3$) balance quality and performance [S1, unofficial].
*   **Boundaries:** The textures are designed to be tiling, ensuring seamless repetition across the sky domain [S1, unofficial].
*   **Degenerate Input:** If the coverage signal is 0, the cloud density is 0. If the height signal is not properly normalized, cloud types may not blend correctly [S1, unofficial].

## Operating characteristics
*   **Performance:** Renders in approximately 2 milliseconds per frame [S1, unofficial].
*   **Memory Usage:** The system consumes 20 MB of RAM [S1, unofficial].
*   **Strengths:**
    *   Real-time performance suitable for modern games.
    *   Supports inter-cloud shadowing and temporal evolution.
    *   Generates realistic, billowy cloud shapes that imply motion.
    *   Low memory footprint compared to asset-based solutions.
*   **Limitations:**
    *   Relies on pre-computed textures, limiting on-the-fly generation of arbitrary cloud structures.
    *   Complexity of the noise combination (Perlin-Worley) may be difficult to tune for specific artistic styles.
*   **When to Use:** Real-time volumetric cloud rendering where performance and memory are constrained, and realistic, evolving cloud shapes are required.
*   **When Not to Use:** Offline rendering where quality is paramount and performance is not a constraint, or when simple, static cloud layers are sufficient.

## Background
The system was developed for *Horizon Zero Dawn* by Guerrilla Games, with significant contributions from Han Vos [S1, unofficial]. It was presented at the Advances in Real-Time Rendering (ARTR) 2015 conference [S1, unofficial]. The development was driven by the need to replace asset-based cloud solutions (billboards, sky domes) which could not support inter-cloud shadowing, evolution, or overhead clouds, and suffered from high memory usage and overdraw [S1, unofficial]. The core noise technique, Worley noise, was introduced by Steven Worley in 1996, originally for caustics and water effects [S1, unofficial]. The "Perlin-Worley" hybrid was developed to combine the connectedness of Perlin noise with the packed, billowy shapes of inverted Worley noise, addressing the lack of large governing shapes in standard fBm approaches [S1, unofficial].

## Sources
[S1] The Real-time Volumetric Cloudscapes of Horizon, https://advances.realtimerendering.com/s2015/The%20Real-time%20Volumetric%20Cloudscapes%20of%20Horizon%20-%20Zero%20Dawn%20-%20ARTR.pdf, fetched 2026-10-29 (unofficial)
