# Fractal Simplex Noise

Also called: Octaved Simplex Noise, Simplex Noise FBM

## Overview
Fractal Simplex Noise generates multi-scale procedural volume data by summing multiple octaves of Simplex Noise, building on [Fractal Brownian Motion (fBm) via Octave Summation](16998-fractal-brownian-motion-fbm-via-octave-summation.md). It differs from standard fBm by using Simplex Noise as the base generator, which offers superior gradient continuity and reduced directional artifacts compared to Perlin noise. The method terminates early when the feature size becomes smaller than one pixel, determined by the inverse of the derivative, optimizing performance for high-frequency details that are visually insignificant [S1, unofficial].

## How it works
**Inputs:**
- `pos`: Input 3D position vector ($x, y, z$).
- `scale`: Initial frequency scale (inverse of wavelength).
- `seed`: Random seed for the Simplex Noise generator.
- `du`: Derivative value (pixel scale threshold).
- `n`: Maximum number of octaves.
- `lacunarity`: Factor by which scale increases per octave.
- `decay`: Factor by which amplitude decreases per octave.

**Outputs:**
- `result`: The accumulated fractal noise value.

**Data Structures:**
- `acc`: Floating-point accumulator, initialized to `0.0`.
- `amp`: Floating-point amplitude, initialized to `1.0`.
- `rdu`: Reciprocal derivative, calculated as `1.0 / du`.

**Algorithm:**
1. Initialize `acc = 0.0`, `amp = 1.0`, and `rdu = 1.0 / du`.
2. Loop for `i` from `0` to `n - 1`:
   a. Calculate the scaled position: `scaled_pos = pos * scale`.
   b. Compute the base noise value: `val = simplexNoise(scaled_pos, 1.0, seed * (i + 1))`.
   c. Add the weighted noise to the accumulator: `acc += val * amp`.
   d. Update parameters for the next octave:
      - `scale *= lacunarity`
      - `amp *= decay`
   e. Check termination condition: If `scale > rdu`, break the loop.
3. Return `acc`.

**Formulas:**
The accumulated value $A$ is given by:
$$ A = \sum_{i=0}^{k-1} \text{Simplex}(\mathbf{p} \cdot \lambda^i, \mathbf{s}_i) \cdot \gamma^i $$
Where:
- $\mathbf{p}$ is the input position.
- $\lambda$ is the lacunarity.
- $\gamma$ is the decay.
- $\mathbf{s}_i$ is the seed for octave $i$, typically $\text{seed} \times (i + 1)$.
- $k$ is the number of octaves actually computed, determined by $\lambda^k \le \frac{1}{du}$.

**Parameters:**
- `lacunarity`: Typically `2.0` [S1, unofficial].
- `decay`: Typically `0.5` [S1, unofficial].
- `du`: Determined by the rendering resolution and camera distance; represents the smallest meaningful feature size [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** If `du` is `0`, `rdu` becomes infinite, potentially causing an infinite loop if `n` is not strictly enforced. Guard against `du == 0` [G].
- **Seed Collision:** Using `seed * (i + 1)` can lead to correlated noise if the seed has specific bit patterns. Ensure the seed is sufficiently random [G].
- **Precision Loss:** At high octaves, `amp` may underflow to zero. The loop termination condition `scale > rdu` helps mitigate this by stopping before extreme frequencies where precision issues dominate [S1, unofficial].

## Subtopics

### Absolute Value Simplex fBm
A variant where the absolute value of each octave's Simplex noise is taken before summation. This creates a "ridged" appearance, often used for terrain or marble-like textures. The loop adds `abs(simplexNoise(...)) * amp * 0.35f` to the accumulator. After summation, the result is mapped back to the standard `[-1, 1]` range using a `mapToSigned` function [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(k)$, where $k$ is the number of octaves. $k$ is typically small (4-8) due to the early termination condition based on `du`.
- **Memory Complexity:** $O(1)$ auxiliary space, excluding the output buffer.
- **Performance:** Faster than fixed-octave fBm in high-resolution scenes because it skips unnecessary high-frequency computations. The Simplex Noise base is slightly more expensive than 2D Perlin but offers better 3D quality.
- **Strengths:** Adaptive detail level; reduced directional artifacts compared to grid-based noise; efficient for ray-marched volumes.
- **Limitations:** The early termination logic requires accurate derivative estimation (`du`); incorrect `du` can lead to aliasing or premature cutoff.

## Background
Fractal Simplex Noise is an application of fractal Brownian motion principles to Simplex Noise, a gradient noise function introduced by Ken Perlin. The specific implementation details regarding derivative-based termination and seed multiplication are documented in the unofficial repository `voxelized3d` [S1, unofficial].

## Sources
[S1] Russoul/voxelized3d, https://github.com/Russoul/voxelized3d, fetched 2026-10-26 (unofficial)
