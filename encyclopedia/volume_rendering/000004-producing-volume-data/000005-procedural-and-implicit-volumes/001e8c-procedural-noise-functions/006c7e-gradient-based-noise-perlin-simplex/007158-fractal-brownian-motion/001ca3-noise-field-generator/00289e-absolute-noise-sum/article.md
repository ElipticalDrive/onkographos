# Absolute Noise Sum

Also called: fractal abs sum

## Overview
Absolute Noise Sum is a fractal noise technique that sums the absolute values of noise function outputs across multiple octaves [S1, unofficial]. Unlike standard Fractal Brownian Motion (fBm) which preserves the signed nature of the noise, this method produces a scalar field with a range of $[0.0, \infty)$ [S1, unofficial]. This characteristic creates "cloudy" or "rough" textures, distinguishing it from its siblings like [FBM Combinations](3962-fbm-combinations.md) and [Turbulence](17000-turbulence.md) by eliminating negative values before accumulation [S1, unofficial].

## How it works
**Inputs:**
- $p$: 3D input coordinate (vector).
- $N_{oct}$: Number of octaves (integer).
- $f_0$: Initial frequency (scalar).
- $g_0$: Initial amplitude (scalar, typically $1.0$) [S1, unofficial].
- $l$: Lacunarity (scalar, typically $2.0$) [S1, unofficial].
- `gradient_noise`: The underlying gradient-based noise function (e.g., Simplex or Perlin) returning a signed value [S1, unofficial].

**Outputs:**
- $S$: The accumulated absolute noise sum (scalar in $[0, \infty)$) [S1, unofficial].

**Data Structures:**
- Standard floating-point accumulators suffice [G].

**Algorithm:**
1. Initialize the sum accumulator $S = 0$ [S1, unofficial].
2. Initialize the current frequency $f = f_0$ [S1, unofficial].
3. Initialize the current amplitude $a = g_0$ [S1, unofficial].
4. Loop for $i$ from $0$ to $N_{oct} - 1$:
   a. Evaluate the noise function at the scaled coordinate: $v = \text{gradient\_noise}(p \cdot f)$ [S1, unofficial].
   b. Add the weighted absolute noise to the sum: $S \leftarrow S + |v| \cdot a$ [S1, unofficial].
   c. Update frequency: $f \leftarrow f \cdot l$ [S1, unofficial].
   d. Update amplitude: $a \leftarrow a \cdot g$ [S1, unofficial].
5. Return $S$ [S1, unofficial].

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
    sum_val = 0.0
    freq = frequency
    amp = 1.0  # Initial amplitude as per source [S1, unofficial]
    
    for _ in range(octaves):
        sum_val += abs(gradient_noise((p[0] * freq, p[1] * freq, p[2] * freq))) * amp
        freq *= lacunarity
        amp *= gain
        
    return sum_val

def gradient_noise(p):
    # Standard gradient noise evaluation (e.g., Simplex/Perlin)
    # Implementation provided by parent article [1302-noise-field-generator.md]
    # Returns a value in [-1, 1] or [0, 1] depending on implementation.
    x, y, z = p
    n = x * 127.1 + y * 311.7 + z * 74.7
    n = (n & 0x7fffffff) / 2147483647.0
    return n * 2.0 - 1.0
```

**Parameters and Typical Values:**
- Initial Amplitude: $1.0$ [S1, unofficial]. This differs from standard fBm where the first octave often has amplitude $0.5$ or $1.0$ depending on normalization, but the key distinction is the absolute value operation [S1, unofficial].
- Lacunarity: $2.0$ [S1, unofficial].
- Gain: Multiplicative factor per octave [S1, unofficial]. In many implementations, this is $< 1.0$ to ensure convergence, but the source notes only specify the initial amplitude is $1.0$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Summing many octaves with high gain can lead to overflow if not normalized [G].
- **Boundary Conditions:** The noise function `gradient_noise` must handle arbitrary floating-point coordinates [G].
- **Degenerate Input:** If `octaves` is 0, the sum is 0 [G].

## Subtopics

### [Perlin Absolute Noise (Turbulence)](18200-perlin-absolute-noise-turbulence.md)
A variant of fBm using Perlin noise where the absolute value of each octave is taken before summing, creating a "billowy" or "cloudy" appearance distinct from the smoother signed sums of standard fBm. It builds on [Absolute Noise Sum](4319-absolute-noise-sum.md) by specifically utilizing Perlin's gradient noise as the base generator.

## Operating characteristics
**Time Complexity:** $O(N_{oct})$ per sample, where $N_{oct}$ is the number of octaves [G]. Each octave requires one noise evaluation and basic arithmetic [G].
**Memory Complexity:** $O(1)$ auxiliary space [G].
**Performance:** The method is computationally lightweight compared to domain-warped variants [4493-domain-warped-fbm.md] as it does not require secondary noise evaluations for coordinate perturbation [G]. However, it is more expensive than a single octave of noise [G].
**Strengths:** Produces natural-looking "cloudy" or "rough" textures with a positive range [S1, unofficial]. Simple to implement [G].
**Limitations:** The output range is unbounded above if gain $\ge 1$, requiring manual normalization for consistent rendering [G]. It lacks the directional bias of standard fBm, which may be undesirable for terrain [4214-terrain-heightmap-generation-via-noise.md] [G].
**When to use:** When a positive-only, rough texture is desired, such as for cloud density or surface roughness maps [G].
**When not to use:** When signed displacement is required (use standard fBm) or when directional features are needed (use Domain Warped FBM) [G].

## Background
The method is described in the context of GPU-based Marching Cubes implementations for generating procedural voxel data [S1, unofficial]. The specific implementation details are found in open-source repositories [S1, unofficial].

## Sources
[S1] Scrawk/Marching-Cubes-On-The-GPU, https://github.com/Scrawk/Marching-Cubes-On-The-GPU, fetched 2026-10-26 (unofficial)
