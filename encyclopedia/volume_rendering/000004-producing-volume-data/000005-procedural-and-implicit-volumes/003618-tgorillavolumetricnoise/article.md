# TGorillaVolumetricNoise

Also called: Volumetric Noise, cloud / smoke / fog body

## Overview
TGorillaVolumetricNoise is a procedural volume component that generates a 3D texture using fractal Brownian motion (FBM) Perlin noise and renders it as a physically based emission-absorption medium [S1]. It solves the problem of creating believable, fly-through clouds, smoke, or fog without requiring external data files or custom shader code [S1]. It distinguishes itself from sibling methods by baking a smooth 0..1 density gradient into the voxel data while applying opacity via a render-time Beer-Lambert blend, thereby avoiding the terracing artifacts common in naive voxel density approaches [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Parent viewport, size (default 1x1x1), resolution (default 64³), and shaping parameters (Frequency, Octaves, Persistence, Lacunarity, Coverage, EdgeFalloff, Billowy, Seed) [S1].
*   **Outputs:** A 3D texture (volume data) uploaded to the GPU and rendered as a volumetric mesh [S1].

**Data Structures:**
The core data structure is a 3D texture (volume) where each voxel stores a single floating-point value representing a smooth density gradient in the range [0, 1] [S1]. This value is *not* the final opacity; it is a shape mask [S1].

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
1.  **Initialization:** Create the instance, set the parent, size, and assign FrontFaceRenderPass and BackFaceRenderPass [S1].
2.  **Noise Generation:** For each voxel in the 3D grid:
    a.  Calculate the base Perlin noise value at the voxel's position (scaled by `Frequency` and shifted by `Offset`) [S1].
    b.  Apply FBM summation: Sum the noise values across `Octaves`. For each octave $i$, multiply the amplitude by `Persistence`^$i$ and the frequency by `Lacunarity`^$i$ [S1].
    c.  If `Billowy` is True, take the absolute value of the resulting FBM sum [S1].
3.  **Shaping:**
    a.  Apply `Coverage`: Use the `Coverage` value as a soft threshold to carve the noise field into separate puffs [S1].
    b.  Apply `EdgeFalloff`: Fade the density radially toward the volume border using `EdgeFalloff` to create a rounded blob shape [S1].
    c.  Clamp the result to the range [0, 1] [S1].
4.  **Storage:** Store the resulting value in the 3D texture [S1].
5.  **Rendering:** During ray marching, use the stored voxel value as a density mask. Apply the `Density` parameter as the absorption coefficient in the Beer-Lambert law to compute opacity and color at render time [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_volumetric_noise(width, height, depth, frequency, octaves, persistence, lacunarity, coverage, edge_falloff, billowy, seed):
    # Initialize 3D texture
    volume = new float[width][height][depth]
    
    # Initialize Perlin noise generator with seed
    perlin = PerlinNoise(seed)
    
    for x in range(width):
        for y in range(height):
            for z in range(depth):
                # Normalize coordinates to [0, 1] range relative to size
                nx = x / width
                ny = y / height
                nz = z / depth
                
                # Calculate base noise value
                # Note: Frequency scales the input coordinates
                base_noise = perlin.noise(nx * frequency, ny * frequency, nz * frequency)
                
                # FBM Summation
                fbm_value = 0.0
                amplitude = 1.0
                current_frequency = 1.0
                
                for i in range(octaves):
                    fbm_value += amplitude * perlin.noise(
                        nx * frequency * current_frequency,
                        ny * frequency * current_frequency,
                        nz * frequency * current_frequency
                    )
                    amplitude *= persistence
                    current_frequency *= lacunarity
                
                # Billowy shaping
                if billowy:
                    fbm_value = abs(fbm_value)
                
                # Coverage shaping (soft threshold)
                # The documentation implies Coverage acts as a threshold/carver.
                # A common implementation for "soft threshold" is a smoothstep or clamp.
                # Based on "carve the body into separate puffs", we assume a threshold application.
                # If Coverage < 1, values below (1-Coverage) or similar might be zeroed.
                # However, the text says "Coverage as a soft threshold".
                # We implement a simple linear falloff from Coverage downwards for clarity,
                # or assume the noise value is compared against Coverage.
                # Given "Value 1 keeps the whole body; lower values carve", 
                # we interpret this as: density = max(0, fbm_value - (1 - coverage)) * scale?
                # Or simply: if fbm_value < coverage, reduce density?
                # The most robust interpretation of "soft threshold" in this context is often:
                # density = smoothstep(coverage - width, coverage + width, fbm_value)
                # But without specific width, we use a direct comparison or linear interpolation.
                # Let's assume a linear falloff below Coverage for simplicity, or just clamp.
                # Actually, "carve" suggests zeroing out low values.
                # Let's use a simple threshold: if fbm_value < (1 - coverage), set to 0?
                # No, "soft" implies smoothness.
                # Let's assume the noise value is multiplied by a factor derived from Coverage,
                # or that Coverage is a cutoff.
                # Given the ambiguity, we stick to the core FBM and apply EdgeFalloff.
                # The "Coverage" parameter is described as "Cloud coverage 0-1".
                # We will apply it as a multiplier to the noise amplitude or a threshold.
                # Let's assume it acts as a threshold for the noise value to be considered "cloud".
                # density = max(0.0, fbm_value - (1.0 - coverage))
                # This creates a "carved" look.
                
                density = max(0.0, fbm_value - (1.0 - coverage))
                
                # Edge Falloff
                # Calculate distance from center (0.5, 0.5, 0.5)
                dist = sqrt((nx - 0.5)**2 + (ny - 0.5)**2 + (nz - 0.5)**2)
                # Max distance in normalized space is sqrt(0.75) ~ 0.866
                # Normalize dist to [0, 1] roughly
                max_dist = 0.866
                normalized_dist = dist / max_dist
                
                # Fade out near the border
                # EdgeFalloff 0 = no fade, 1 = full fade at border
                edge_factor = 1.0 - (normalized_dist * edge_falloff)
                edge_factor = max(0.0, edge_factor)
                
                density *= edge_factor
                
                # Clamp to [0, 1]
                density = min(1.0, max(0.0, density))
                
                volume[x][y][z] = density
                
    return volume
```

**Edge Cases and Pitfalls:**
*   **Terracing Artifacts:** Naive approaches that bake final density/opacity into voxel data and use linear blending produce hard saturation "shells" or step terraces when viewed from inside. This is solved by storing a smooth 0..1 gradient and applying opacity at render time via emission-absorption blend [S1].
*   **Performance:** `Regenerate` re-samples the whole volume on the CPU and is expensive. For animation, use a smaller resolution (e.g., 32³) and update on a timer (~100 ms) rather than every frame [S1].
*   **Lighting:** `UseLighting` should remain False for standard volumetric noise. Lit clouds require a separate "light-marching" feature [S1].

## Operating characteristics
*   **Time Complexity:** Generation is $O(N)$ where $N$ is the number of voxels ($W \times H \times D$). Each voxel requires $O(\text{Octaves})$ noise evaluations. Rendering is standard volumetric ray marching.
*   **Memory Complexity:** $O(W \times H \times D)$ for the 3D texture. Default resolution is 64³, which can be reduced to 32³ for performance [S1].
*   **Measured Performance:** Generation is CPU-bound. Animation requires careful management of update frequency to avoid stuttering [S1].
*   **Strengths:** Resolution-independent density appearance (a 32³ cloud looks as dense as a 64³ cloud with the same `Density` parameter); no external data files; fly-through capability [S1].
*   **Limitations:** CPU-bound regeneration makes high-resolution animation expensive; lacks built-in lighting (requires separate feature) [S1].
*   **When to use:** For static or slowly drifting clouds, smoke, or fog where performance allows for CPU regeneration or lower resolutions are acceptable [S1].
*   **When not to use:** For high-frequency animated smoke requiring high resolution, or for lit volumetric clouds (use light-marching instead) [S1].

## Background
TGorillaVolumetricNoise is part of the Gorilla3D framework [S1]. It is designed to provide a "ready-to-use" cloud/smoke/fog body with a handful of properties, abstracting away the need for shader code or data files [S1]. It builds on the principles of [Procedural and implicit volumes](13-procedural-and-implicit-volumes.md) and [Volumetric Mesh](11-producing-volume-data.md) [S1].

## Sources
[S1] Volumetric Noise - Gorilla3D Framework - Documentation, https://gorilla3d.de/doc/v3/version/v1.4/page/Core/Volumetric/Volumetric-Noise.md, fetched 2026-10-29
