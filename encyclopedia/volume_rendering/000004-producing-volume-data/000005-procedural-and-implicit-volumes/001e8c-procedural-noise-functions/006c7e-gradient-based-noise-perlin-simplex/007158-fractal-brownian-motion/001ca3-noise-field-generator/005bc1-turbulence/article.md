# Turbulence

Also called: Turbulent Noise

## Overview
Turbulence is a noise basis function that creates the visual impression of Brownian motion or turbulent flow by summing absolute values of noise at decreasing frequencies [S2, book]. It differs from standard Fractal Brownian Motion (FBM) by applying the absolute value operator to each octave, which introduces discontinuities in the gradient and reflects the underlying noise vectors [S2, book].

## How it works
**Inputs and Outputs**
- Input: A 3D position vector $\mathbf{p}$ ($\mathbb{R}^3$) [G].
- Output: A scalar value representing the accumulated turbulent noise magnitude [S1, unofficial].

**Data Structures**
The method requires a base noise function $\mathcal{N}$ that returns a signed scalar value [G].

**Parameters**
- `octives`: Integer constant, typically 4. Defines the number of octaves to accumulate [S1, unofficial].
- `lacunarity`: Float constant, typically 2.0. Multiplier for frequency in each subsequent octave [S1, unofficial].
- `gain`: Float constant, typically 0.5. Multiplier for amplitude in each subsequent octave [S1, unofficial].

**Algorithm Steps**
1. Initialize `sum = 0.0`, `freq = 1.0`, and `amp = 1.0` [S1, unofficial].
2. Loop `i` from 0 to `octives - 1`:
   a. Evaluate the base noise function at the scaled position: `n = \mathcal{N}(\mathbf{p} \cdot \text{freq})` [S1, unofficial].
   b. Take the absolute value of the noise result: `|n|` [S1, unofficial].
   c. Add the weighted absolute noise to the sum: `sum += |n| \cdot \text{amp}` [S1, unofficial].
   d. Update frequency: `freq *= lacunarity` [S1, unofficial].
   e. Update amplitude: `amp *= gain` [S1, unofficial].
3. Return `sum` [S1, unofficial].

**Formula**
The mathematical definition for $N$ octaves is:
$$ T(\mathbf{p}) = \sum_{i=0}^{N-1} | \mathcal{N}( \mathbf{p} \cdot \lambda^i ) | \cdot g^i $$
Where:
- $\mathbf{p}$ is the input position [G].
- $\mathcal{N}$ is the base noise function returning a signed scalar [G].
- $\lambda$ is the lacunarity (typically 2.0) [S1, unofficial].
- $g$ is the gain (typically 0.5) [S1, unofficial].
- $N$ is the number of octaves (typically 4) [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def turbulence(pos, octives=4, lacunarity=2.0, gain=0.5):
    sum_val = 0.0
    freq = 1.0
    amp = 1.0
    for i in range(octives):
        # Evaluate base noise function at scaled position; returns signed scalar [G]
        # The sources do not specify the exact base noise implementation [S1, unofficial]
        noise_val = base_noise(pos * freq)
        sum_val += abs(noise_val) * amp
        freq *= lacunarity
        amp *= gain
    return sum_val
```

**Edge Cases and Pitfalls**
- **Discontinuities**: The use of `abs()` creates sharp corners in the noise field where the original noise crosses zero, which is intentional for the "turbulent" look [S2, book].
- **Range**: The output range is not bounded to $[0, 1]$ or $[-1, 1]$ like the base noise, and grows with the number of octaves [G].
- **Precision**: Accumulating many octaves with high lacunarity can lead to precision loss in the lower bits if fixed-point arithmetic is used; floating-point is recommended [G].

## Subtopics

### [FBM Combinations](3962-fbm-combinations.md)
Combines multiple FBM layers with different weights and offsets [G]. Use when smooth, continuous fractal detail is needed [G].

### [Terrain Heightmap Generation via Noise](4214-terrain-heightmap-generation-via-noise.md)
Uses noise to generate elevation data [G]. Use for procedural terrain [G].

### [Domain Warped FBM](4493-domain-warped-fbm.md)
Distorts the input coordinates of FBM using another noise field [G]. Use for organic, swirling patterns like smoke or marble [G].

## Operating characteristics
- **Time Complexity**: $O(N)$, where $N$ is the number of octaves [S1, unofficial]. Each octave requires one evaluation of the base noise function [S1, unofficial].
- **Memory Complexity**: $O(1)$ auxiliary space [G].
- **Performance**: The cost is dominated by the base noise function evaluation [G].
- **Strengths**: Creates visually distinct, fragmented textures that mimic turbulent fluids or rough surfaces [S2, book]. Simple to implement [G].
- **Limitations**: The discontinuities from `abs()` can look unnatural if not tuned correctly [S2, book]. It lacks the smoothness of standard FBM [G].
- **When to use**: When simulating solid textures with a "carved" appearance, or when the visual style requires sharp transitions in noise gradients [S2, book].
- **When not to use**: When smooth, continuous variation is required (use standard FBM instead) [G].

## Background
The turbulence basis function was introduced in the context of solid texturing for volume rendering [S2, book]. It is described in the book *Volume Visualization and Volume Graphics* [S2, book], where it is noted for giving the impression of Brownian motion. The specific implementation using `mnoise` (a modified noise function) is found in the *VolumetricFire* project [S1, unofficial]. The technique relies on the self-similar $1/f$ pattern introduced by summing noise at decreasing frequencies, with the absolute value function reflecting the gradient vectors to create discontinuities [S2, book].

## Sources
[S1] yomotsu/VolumetricFire, https://github.com/yomotsu/VolumetricFire, fetched 2026-10-27 (unofficial)
[S2] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
