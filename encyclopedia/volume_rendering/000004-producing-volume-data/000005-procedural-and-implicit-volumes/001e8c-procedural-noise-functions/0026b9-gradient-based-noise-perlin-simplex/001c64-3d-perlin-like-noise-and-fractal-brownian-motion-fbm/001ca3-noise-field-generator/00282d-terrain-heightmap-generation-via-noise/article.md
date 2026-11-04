# Terrain Heightmap Generation via Noise

Also called: GetHeightAtWorldLocation, Heightmap Generation via Noise, Perlin Noise Height Generation, Procedural terrain heightmap, World Height Noise

## Overview
This method generates a terrain heightmap by converting 2D world coordinates into a single height value using scaled fractal noise, remapped to a specific vertical range [S1, unofficial]. It differs from its parent [Noise Field Generator](1302-noise-field-generator.md) by operating directly on 2D world coordinates rather than sampling a 3D scalar field [S1, unofficial].

## How it works
**Inputs:**
- `WorldX`, `WorldY`: The 2D world-space coordinates for which to compute height [S1, unofficial].
- `Settings`: A configuration struct containing noise parameters (`Seed`, `Scale`, `Octaves`, `Persistence`, `Lacunarity`, `HeightScale`, `BaseHeight`) [S1, unofficial].

**Outputs:**
- `Height`: A single floating-point value representing the terrain elevation at `(WorldX, WorldY)` [S1, unofficial].

**Data Structures:**
The `Settings` struct contains the following fields with exact defaults and ranges from the evidence [S1, unofficial]:
- `Seed` (int32): Random seed for noise generation. Default: 1337.
- `Scale` (float): World units per noise cell. Controls smoothness. Range: 1000–5000. Default: 2000.0f.
- `Octaves` (int32): Number of noise layers summed. Controls detail. Range: 1–8. Default: 4.
- `Persistence` (float): Amplitude falloff per octave. Controls roughness. Range: 0.0–1.0. Default: 0.5f.
- `Lacunarity` (float): Frequency multiplier per octave. Standard choice: 2.0. Default: 2.0f.
- `HeightScale` (float): Final output amplitude. The output is remapped to `[BaseHeight - HeightScale, BaseHeight + HeightScale]`. Default: 500.0f.
- `BaseHeight` (float): Final output offset. Default: 0.0f.

**Algorithm Steps:**

1.  **Scale Normalization**: Compute the normalized noise coordinates by dividing world coordinates by the scale factor. A minimum scale floor is applied to prevent division by zero.
    $$
    Scale_{eff} = \max(Settings.Scale, \epsilon)
    $$
    $$
    NoiseX = \frac{WorldX}{Scale_{eff}}, \quad NoiseY = \frac{WorldY}{Scale_{eff}}
    $$
    Where $\epsilon$ is a small number (e.g., $10^{-6}$) [G].

2.  **Fractal Noise Evaluation**: Compute the raw fractal Brownian motion (FBM) value using 2D Perlin noise. This involves summing multiple octaves of noise with increasing frequency and decreasing amplitude.
    $$
    Noise = \text{FractalPerlin2D}(NoiseX, NoiseY, Settings)
    $$
    The result is normalized to approximately $[-1, 1]$ [S1, unofficial].

3.  **Remapping**: Map the normalized noise value to the target vertical range defined by `BaseHeight` and `HeightScale`.
    $$
    Height = BaseHeight + (Noise \times HeightScale)
    $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def GenerateHeight(WorldX: float, WorldY: float, Settings: NoiseSettings) -> float:
    # 1. Scale Normalization
    scale = max(Settings.Scale, 1e-6) # Guard against division by zero [G]
    noise_x = WorldX / scale
    noise_y = WorldY / scale

    # 2. Fractal Noise Evaluation
    raw_noise = FractalPerlin2D(noise_x, noise_y, Settings)

    # 3. Remapping
    height = Settings.BaseHeight + (raw_noise * Settings.HeightScale)
    return height

def FractalPerlin2D(X: float, Y: float, Settings: NoiseSettings) -> float:
    total = 0.0
    frequency = 1.0
    amplitude = 1.0
    max_value = 0.0
    octaves = max(1, min(Settings.Octaves, 8)) # Clamp to [1, 8] [S1, unofficial]

    for _ in range(octaves):
        sample = RawPerlin2D(X * frequency, Y * frequency, Settings.Seed)
        total += sample * amplitude
        max_value += amplitude
        amplitude *= Settings.Persistence
        frequency *= Settings.Lacunarity

    return total / max_value if max_value > 0.0 else 0.0

def RawPerlin2D(X: float, Y: float, Seed: int) -> float:
    # Standard 2D Perlin noise [G]
    xi = math.floor(X) & 255
    yi = math.floor(Y) & 255
    xf = X - math.floor(X)
    yf = Y - math.floor(Y)
    u = fade(xf)
    v = fade(yf)

    # Permutation table P initialized from Seed [G]
    aa = P[P[xi] + yi]
    ab = P[P[xi] + yi + 1]
    ba = P[P[xi + 1] + yi]
    bb = P[P[xi + 1] + yi + 1]

    x1 = lerp(u, grad(aa, xf, yf), grad(ba, xf - 1.0, yf))
    x2 = lerp(u, grad(ab, xf, yf - 1.0), grad(bb, xf - 1.0, yf - 1.0))
    return lerp(v, x1, x2)

def fade(t: float) -> float:
    return t * t * t * (t * (t * 6.0 - 15.0) + 10.0)

def lerp(t: float, a: float, b: float) -> float:
    return a + t * (b - a)

def grad(hash_val: int, x: float, y: float) -> float:
    h = hash_val & 3
    if h == 0: return x + y
    elif h == 1: return -x + y
    elif h == 2: return x - y
    else: return -x - y
```

**Edge Cases and Pitfalls:**
- **Division by Zero**: If `Settings.Scale` is 0, the normalization step fails. A minimum epsilon floor is required [G].
- **Precision Loss**: At very large world coordinates, floating-point precision in `WorldX/Scale` may cause noise artifacts. Wrapping coordinates or using double precision may be necessary for infinite worlds [G].
- **Discontinuities**: If the permutation table is not properly seeded or if `Seed` is not handled consistently across chunks, seams may appear at chunk boundaries [G].

## Subtopics

### Fractal Brownian Motion (FBM)
The underlying noise summation technique [S1, unofficial]. It creates detail by layering noise at different frequencies [S1, unofficial]. Choose this when you need multi-scale terrain features (large mountains + small rocks) [G]. See [3D Perlin-like Noise and Fractal Brownian Motion (FBM)](1303-3d-perlin-like-noise-and-fbm.md).

### Perlin Noise
The base gradient noise function [S1, unofficial]. It provides smooth, continuous interpolation [G]. Choose this over value noise for smoother terrain slopes [G]. See [Gradient-based Noise (Perlin/Simplex)](1301-gradient-based-noise-perlin-simplex.md).

## Operating characteristics
- **Time Complexity**: $O(O)$, where $O$ is the number of octaves [G].
- **Memory Complexity**: $O(1)$ per query [G].
- **Measured Performance**: Not given in the sources.
- **Scaling**: Not given in the sources.
- **Strengths**: Deterministic (given same seed), continuous, easy to tune via parameters [G].
- **Limitations**: 2D projection loses vertical variation details that 3D noise might provide. Not suitable for caves or overhangs without additional 3D processing [G].

## Background
This method is derived from procedural generation techniques used in voxel games [G]. The implementation is found in the Manolo94/UE5MinecraftClone repository [S1, unofficial]. It builds on the [Noise Field Generator](1302-noise-field-generator.md) by simplifying the 3D field into a 2D heightmap [S1, unofficial].

## Sources
[S1] Manolo94/UE5MinecraftClone, https://github.com/Manolo94/UE5MinecraftClone, fetched 2026-10-26 (unofficial)
