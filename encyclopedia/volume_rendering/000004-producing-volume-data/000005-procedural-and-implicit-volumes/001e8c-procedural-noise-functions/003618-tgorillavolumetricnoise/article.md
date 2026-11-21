# TGorillaVolumetricNoise

Also called: Volumetric Noise, cloud / smoke / fog body

## Overview
TGorillaVolumetricNoise is a procedural volume component that generates a 3D texture using fractal Brownian motion (FBM) Perlin noise and renders it as a physically based emission-absorption medium [S1]. It solves the problem of creating believable, fly-through clouds, smoke, or fog without requiring external data files or custom shader code [S1]. It distinguishes itself from sibling methods by baking a smooth 0..1 density gradient into the voxel data while applying opacity via a render-time Beer-Lambert blend, thereby avoiding the terracing artifacts common in naive voxel density approaches [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Parent viewport, size (default 1×1×1), resolution (default 64³), and shaping parameters (Frequency, Octaves, Persistence, Lacunarity, Coverage, EdgeFalloff, Billowy, Seed) [S1].
*   **Outputs:** A 3D texture (volume data) uploaded to the GPU and rendered as a volumetric mesh [S1].

**Data Structures:**
The core data structure is a 3D texture where each voxel stores a single floating-point value representing a smooth density gradient in the range [0, 1] [S1]. This value is *not* the final opacity; it acts as a shape mask [S1].

**Parameters:**
*   `Frequency` (Single): Base noise scale. Higher values produce smaller, more numerous noise cycles [S1].
*   `Octaves` (Integer): Number of FBM octaves summed. More octaves add finer detail but increase generation time [S1].
*   `Persistence` (Single): Amplitude falloff per octave (range 0–1). Lower values yield smoother noise; higher values yield rougher noise [S1].
*   `Lacunarity` (Single): Frequency growth per octave (typically ~2.0) [S1].
*   `Coverage` (Single): Soft threshold (0–1). Value 1 keeps the whole body; lower values carve the body into separate puffs [S1].
*   `EdgeFalloff` (Single): Radial fade (0–1) toward the volume border to create a rounded blob shape instead of a filled cube [S1].
*   `Billowy` (Boolean): If True, applies `abs()` to the noise for a puffy cumulus/smoke look; if False, produces smooth haze [S1].
*   `Density` (Single): Overall opacity/absorption strength. This is *not* baked into the voxel data but applied at render time [S1].
*   `RenderInside` (Boolean): Default True. Allows the camera to move inside the cloud with correct compositing [S1].
*   `UseLighting` (Boolean): Default False. Volumetric noise is an emission-absorption medium, not a lit surface [S1].
*   `Offset` (Point3D): Used for animation drift [S1].

**Algorithm Steps:**
1.  **Initialization:** Create the instance, set the parent, size, assign FrontFaceRenderPass and BackFaceRenderPass, allow control, and apply a preset [S1].
2.  **Noise Generation:** For each voxel in the 3D grid, calculate the base Perlin noise value at the voxel's position scaled by `Frequency` and shifted by `Offset` [S1]. Apply FBM summation: sum the noise values across `Octaves`, scaling amplitude by `Persistence` and frequency by `Lacunarity` per octave [S1]. If `Billowy` is True, apply `abs()` to the resulting FBM sum [S1].
3.  **Shaping:** Apply `Coverage` as a soft threshold to carve the noise field into separate puffs [S1]. Apply `EdgeFalloff` as a radial fade toward the volume border to create a rounded blob shape [S1].
4.  **Storage:** Store the resulting smooth 0..1 gradient in the 3D texture; do not bake the final opacity into the voxel data [S1].
5.  **Rendering:** During ray marching, use the stored voxel value as a density mask and apply the `Density` parameter as the absorption coefficient in the Beer-Lambert emission-absorption blend to compute opacity and color at render time [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def generate_volumetric_noise(width, height, depth, frequency, octaves, persistence, lacunarity, coverage, edge_falloff, billowy, seed):
    # [G] Initialize standard 3D Perlin noise generator with seed
    perlin = PerlinNoise(seed)
    volume = [[0.0] * depth for _ in range(height)]

    for y in range(height):
        for x in range(width):
            for z in range(depth):
                nx = x / width
                ny = y / height
                nz = z / depth

                fbm_value = 0.0
                amplitude = 1.0
                current_freq = 1.0

                for i in range(octaves):
                    fbm_value += amplitude * perlin.noise(
                        nx * frequency * current_freq,
                        ny * frequency * current_freq,
                        nz * frequency * current_freq
                    )
                    amplitude *= persistence
                    current_freq *= lacunarity

                if billowy:
                    fbm_value = abs(fbm_value)

                # Coverage as a soft threshold
                density = max(0.0, fbm_value - (1.0 - coverage))

                # EdgeFalloff radial fade
                dist = math.sqrt((nx - 0.5)**2 + (ny - 0.5)**2 + (nz - 0.5)**2)
                max_dist = math.sqrt(0.75)
                normalized_dist = dist / max_dist
                edge_factor = max(0.0, 1.0 - normalized_dist * edge_falloff)

                density *= edge_factor
                volume[y][x][z] = min(1.0, max(0.0, density))

    return volume
```

**Edge Cases and Pitfalls:**
*   **Terracing Artifacts:** Naïve approaches that bake final density/opacity into voxel data and use linear blending produce hard saturation "shells" or step terraces when viewed from inside [S1]. This is solved by storing a smooth 0..1 gradient and applying opacity at render time via emission-absorption blend [S1].
*   **Performance:** `Regenerate` re-samples the whole volume on the CPU and is expensive [S1]. For animation, use a smaller resolution (e.g., 32³) and update on a timer (~100 ms) rather than every frame [S1].
*   **Lighting:** `UseLighting` should remain False for standard volumetric noise [S1]. Lit clouds require a separate "light-marching" feature [S1].

## Operating characteristics
*   **Generation Cost:** Generation is CPU-bound and requires re-sampling the entire volume on the CPU when parameters change or animation drifts [S1].
*   **Resolution Independence:** Density appearance is resolution-independent; a cloud at 32³ looks comparably dense to one at 64³ for the same `Density` value [S1].
*   **Strengths:** No external data files or shader code required; fly-through capability with correct compositing; cheap runtime opacity adjustment via `Density` [S1].
*   **Limitations:** CPU-bound regeneration makes high-resolution animation expensive; lacks built-in lighting [S1].
*   **When to use:** For static or slowly drifting clouds, smoke, or fog where performance allows for CPU regeneration or lower resolutions are acceptable [S1].
*   **When not to use:** For high-frequency animated smoke requiring high resolution, as CPU regeneration becomes too expensive; use a smaller resolution (e.g., 32³) or a different method instead [S1]. For lit volumetric clouds, use light-marching instead [S1].

## Background
TGorillaVolumetricNoise is part of the Gorilla3D framework [S1]. It is designed to provide a "ready-to-use" cloud/smoke/fog body with a handful of properties, abstracting away the need for shader code or data files [S1]. It builds on the principles of [Procedural and implicit volumes](13-procedural-and-implicit-volumes.md) and [Volumetric Mesh](11-producing-volume-data.md) [S1].

## Sources
[S1] Volumetric Noise - Gorilla3D Framework - Documentation, https://gorilla3d.de/doc/v3/version/v1.4/page/Core/Volumetric/Volumetric-Noise.md, fetched 2026-10-29
