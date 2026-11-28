# Fractal Brownian Motion (fBm) via Octave Summation

Also called: Fractal Noise, Fractal Sum, Fractional Brownian Motion, Multi-octave Noise, Octave Noise, Octave Summation

## Overview
Fractal Brownian Motion (fBm) via Octave Summation generates multi-scale procedural volume data by summing multiple octaves of a base noise function, scaling frequency and amplitude per octave to create fractal noise [S2, unofficial]. It differs from its parent [Simplex Noise](978-simplex-noise.md) by accumulating hierarchical detail across scales [S4, unofficial], and from siblings like [Simplex Noise 2D](4155-simplex-noise-2d.md), [Simplex Noise 3D Gradient](3496-simplex-noise-3d-gradient.md), and [Simplex-based Encoding](9582-simplex-based-encoding.md) by operating in 3D volume space with configurable spectral decay [G].

## How it works
**Inputs and Outputs:**
- Input: 3D coordinate $\mathbf{p} = (x, y, z)$, integer `octaves` $N$, `lacunarity` $L$ (frequency multiplier), `gain` $G$ (amplitude decay), and a base noise function $B(\mathbf{p})$ (e.g., Simplex Noise) [S1, unofficial].
- Output: Scalar field value $V \in \mathbb{R}$ [S2, unofficial].

**Data Structures:**
No persistent data structures are required; the method is stateless and computes values on demand [S2, unofficial].

**Algorithm:**
1. Initialize accumulator $V = 0.0$.
2. Initialize current amplitude $A = 1.0$.
3. Initialize current frequency multiplier $F = 1.0$.
4. Loop for $i$ from $0$ to $N-1$:
   a. Scale the input coordinate: $\mathbf{p}_i = \mathbf{p} \times F$.
   b. Evaluate base noise: $n = B(\mathbf{p}_i)$.
   c. Add weighted contribution: $V = V + n \times A$.
   d. Update amplitude: $A = A \times G$.
   e. Update frequency: $F = F \times L$.
5. Return $V$.

**Formula:**
The accumulated value is computed as:
$$V(\mathbf{p}) = \sum_{i=0}^{N-1} G^i \cdot B(\mathbf{p} \cdot L^i)$$
where $N$ is the number of octaves, $L$ is the lacunarity (frequency multiplier), $G$ is the gain (amplitude decay), and $B$ is the base noise function [S1, unofficial]. The theoretical stochastic definition uses the Weierstrass-Mandelbrot function, describing an infinite sum of sine waves with varying frequency, random phase shifts, and amplitude decaying as a function of frequency [S1, unofficial].

**Parameters:**
- `octaves` ($N$): Integer count of noise layers to sum. Higher values increase detail at smaller scales [S2, unofficial].
- `lacunarity` ($L$): Frequency multiplier for each octave. Typically $L=2.0$ [S4, unofficial].
- `gain` ($G$): Amplitude decay factor per octave. Typically $G=0.5$ [S4, unofficial].
- `basis`: The base noise function used (e.g., Simplex Noise) [S1, unofficial].

**Edge Cases and Pitfalls:**
- If $G \ge 1.0$, the signal may diverge or saturate, losing fractal characteristics [G].
- High `octaves` values increase computational cost linearly [S1, unofficial].
- Precision loss may occur if $F$ becomes very large, causing coordinate wrapping or quantization errors in the base noise function [G].
- The term "octave" implies frequency doubling, which is strictly valid only when the frequency is exactly doubled for each octave [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fbm_octave_summation(p, octaves, lacunarity, gain, base_noise):
    value = 0.0
    amplitude = 1.0
    frequency = 1.0
    
    for i in range(octaves):
        scaled_p = (p[0] * frequency, p[1] * frequency, p[2] * frequency)
        noise_val = base_noise(scaled_p[0], scaled_p[1], scaled_p[2])
        value += noise_val * amplitude
        amplitude *= gain
        frequency *= lacunarity
        
    return value
```

## Subtopics

### [Fractal Simplex Noise](18144-fractal-simplex-noise.md)
Fractal Simplex Noise generates multi-scale procedural volume data by summing multiple octaves of Simplex Noise, building on [Fractal Brownian Motion (fBm) via Octave Summation](16998-fractal-brownian-motion-fbm-via-octave-summation.md). The method terminates early when the feature size (scale) becomes smaller than one pixel, determined by the inverse of the derivative `du`, optimizing performance for high-frequency details that are visually insignificant.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of octaves. Each octave requires one evaluation of the base noise function [S1, unofficial].
- **Memory Complexity:** $O(1)$, as no intermediate data structures are stored [S2, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Performance scales linearly with `octaves`. The cost is dominated by the base noise function evaluation [S4, unofficial].
- **Strengths:** Generates natural-looking, multi-scale detail with minimal memory overhead [G].
- **Limitations:** Computationally expensive for high `octaves` counts. Lacks true spectral continuity at octave boundaries if the base noise is not carefully constructed [G].
- **When to use:** Ideal for procedural volume textures, terrain generation, and organic surface details where multi-scale variation is required [S1, unofficial].
- **When not to use:** Avoid when real-time performance is critical and high-frequency detail is not visually necessary, or when strict spectral continuity is required without aliasing artifacts [G].

## Background
Fractal Brownian Motion is a mathematical model used to describe random processes with self-similar properties [G]. In computer graphics, it was adapted to generate procedural noise by summing multiple octaves of a base noise function [S5, unofficial]. The method is widely used in procedural generation for volume rendering and terrain synthesis [S1, unofficial]. Musgrave [1993] notes that the technique acts as a "poor man's Fourier series" that lacks precision but is very fast [S1, unofficial]. The implementation scales the roughness of the noise sum based on altitude to produce realistic terrain features, such as rounded foothills versus jagged peaks [S1, unofficial].

## Sources
[S1] Terrain synthesis using noise, https://trepo.tuni.fi/handle/10024/101043, fetched 2026-10-26 (unofficial)
[S2] Russoul/voxelized3d, https://github.com/Russoul/voxelized3d, fetched 2026-10-26 (unofficial)
[S3] jshrake/lexov, https://github.com/jshrake/lexov, fetched 2026-10-27 (unofficial)
[S4] vittoboa/procedural-terrain-generation, https://github.com/vittoboa/procedural-terrain-generation, fetched 2026-10-27 (unofficial)
[S5] PDF Noise and Procedural Techniques - NVIDIA, https://www.nvidia.com/docs/IO/8228/GDC2003_OGL_ProceduralNoise.pdf, fetched 2026-10-27 (unofficial)
