# Absolute Noise Sum

Also called: fractal abs sum

## Overview
Absolute Noise Sum is a fractal noise technique that sums the absolute values of noise function outputs across multiple octaves [S1, unofficial]. Unlike standard Fractal Brownian Motion (fBm) which preserves the signed nature of the noise, this method produces a scalar field with a range of $[0.0, 1.0]$ [S1, unofficial]. This characteristic creates "cloudy" or "rough" textures, distinguishing it from its siblings like [FBM Combinations](3962-fbm-combinations.md) and [Turbulence](17000-turbulence.md) by eliminating negative values before accumulation [S1, unofficial].

## How it works
**Inputs:**
- $p$: 3D input coordinate (vector).
- $N_{oct}$: Number of octaves (integer).
- $f_0$: Initial frequency (scalar).
- $g_0$: Initial gain/amplitude (scalar, typically $1.0$) [S1, unofficial].
- $l$: Lacunarity (scalar, typically $2.0$) [S1, unofficial].
- $inoise(p)$: The underlying gradient-based noise function (e.g., Simplex or Perlin) returning a value in $[-1, 1]$ or $[0, 1]$ depending on implementation, but treated as a signed magnitude here [S1, unofficial].

**Outputs:**
- $S$: The accumulated absolute noise sum (scalar in $[0, \infty)$, often normalized).

**Data Structures:**
- No complex structures required. Standard floating-point accumulators suffice.

**Algorithm:**
1. Initialize the sum accumulator $S = 0$.
2. Initialize the current frequency $f = f_0$.
3. Initialize the current amplitude $a = g_0$ [S1, unofficial].
4. Loop for $i$ from $0$ to $N_{oct} - 1$:
   a. Evaluate the noise function at the scaled coordinate: $v = inoise(p \cdot f)$.
   b. Take the absolute value: $|v|$.
   c. Add the weighted absolute noise to the sum: $S \leftarrow S + |v| \cdot a$.
   d. Update frequency: $f \leftarrow f \cdot l$.
   e. Update amplitude: $a \leftarrow a \cdot g$ (where $g$ is the gain per octave, often implicit or equal to $1$ if $g_0$ is the only parameter, but standard fBm decay is implied by "Gain" [S1, unofficial]). *Note: The source specifies `amp *= _Gain` [S1, unofficial]. If Gain is not explicitly defined as a decay factor distinct from the initial amplitude, it is treated as the multiplicative factor per octave.*
5. Return $S$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def absolute_noise_sum(p, octaves, frequency, lacunarity, gain):
    """
    Computes the sum of absolute noise values over multiple octaves.
    
    Args:
        p (tuple): 3D input coordinate (x, y, z).
        octaves (int): Number of noise octaves to sum.
        frequency (float): Initial frequency scaling factor.
        lacunarity (float): Frequency multiplier per octave (typically 2.0).
        gain (float): Amplitude multiplier per octave.
        
    Returns:
        float: The accumulated absolute noise sum.
    """
    # Initialize variables
    sum_val = 0.0
    freq = frequency
    amp = 1.0  # Initial amplitude as per source [S1, unofficial]
    
    # Loop over octaves
    for i in range(octaves):
        # Evaluate noise at scaled position
        # inoise is assumed to be the underlying gradient noise function
        # defined in the parent article [Noise Field Generator](1302-noise-field-generator.md)
        noise_val = inoise((p[0] * freq, p[1] * freq, p[2] * freq))
        
        # Take absolute value and add to sum
        sum_val += abs(noise_val) * amp
        
        # Update frequency and amplitude for next octave
        freq *= lacunarity
        amp *= gain
        
    return sum_val
```

**Parameters and Typical Values:**
- Initial Amplitude: $1.0$ [S1, unofficial]. This differs from standard fBm where the first octave often has amplitude $0.5$ or $1.0$ depending on normalization, but the key distinction is the absolute value operation [S1, unofficial].
- Lacunarity: $2.0$ [S1, unofficial].
- Gain: Not explicitly defined as a decay value in the snippet, but `amp *= _Gain` implies a multiplicative factor [S1, unofficial]. In many implementations, this is $< 1.0$ to ensure convergence, but the source notes only specify the initial amplitude is $1.0$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Summing many octaves with high gain can lead to overflow if not normalized. The source does not specify normalization, so the output range is $[0, \infty)$ [S1, unofficial].
- **Boundary Conditions:** The noise function `inoise` must handle arbitrary floating-point coordinates. Standard gradient noise handles this via integer grid hashing [S1, unofficial].
- **Degenerate Input:** If `octaves` is 0, the sum is 0.

## Subtopics

### Perlin Absolute Noise (Turbulence)
A variant of fBm using Perlin noise where the absolute value of each octave is taken before summing, creating a "turbulent" appearance [S2, unofficial]. It differs from the standard Absolute Noise Sum by explicitly mapping the accumulated absolute value back to the standard $[-1, 1]$ range using a `mapToSigned` function [S2, unofficial]. This allows the result to retain signed information after the turbulence effect, unlike the strictly positive output of the base method [S2, unofficial].

## Operating characteristics
**Time Complexity:** $O(N_{oct})$ per sample, where $N_{oct}$ is the number of octaves. Each octave requires one noise evaluation and basic arithmetic.
**Memory Complexity:** $O(1)$ auxiliary space.
**Performance:** The method is computationally lightweight compared to domain-warped variants [4493-domain-warped-fbm.md] as it does not require secondary noise evaluations for coordinate perturbation. However, it is more expensive than a single octave of noise.
**Strengths:** Produces natural-looking "cloudy" or "rough" textures with a bounded positive range [S1, unofficial]. Simple to implement.
**Limitations:** The output range is unbounded above if gain $\ge 1$, requiring manual normalization for consistent rendering. It lacks the directional bias of standard fBm, which may be undesirable for terrain [4214-terrain-heightmap-generation-via-noise.md].
**When to use:** When a positive-only, rough texture is desired, such as for cloud density or surface roughness maps.
**When not to use:** When signed displacement is required (use standard fBm) or when directional features are needed (use Domain Warped FBM).

## Background
The method is described in the context of GPU-based Marching Cubes implementations [S1, unofficial]. It is used to generate procedural voxel data for 3D modeling and rendering [S1, unofficial]. The specific implementation details are found in open-source repositories [S1, unofficial][S2, unofficial].

## Sources
[S1] Scrawk/Marching-Cubes-On-The-GPU, https://github.com/Scrawk/Marching-Cubes-On-The-GPU, fetched 2026-10-26 (unofficial)
[S2] Russoul/voxelized3d, https://github.com/Russoul/voxelized3d, fetched 2026-10-26 (unofficial)
