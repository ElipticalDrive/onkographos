# Polar Modulo

Also called: Angular Modulo, Polar Tiling

## Overview
Polar Modulo repeats a 2D pattern around the origin by a fixed angle determined by the number of repetitions [S1, unofficial]. It builds on [Domain Repetition](1218-domain-repetition.md) by applying modulo arithmetic to the angular coordinate rather than Cartesian coordinates, enabling radial symmetry [S1, unofficial]. Unlike [Space Repetition](1677-space-repetition.md) which tiles in linear space, Polar Modulo creates concentric or angular sectors [S1, unofficial].

## How it works
**Inputs:** A 2D point $p = (x, y)$, and an integer $N$ (number of repetitions) [S1, unofficial].
**Outputs:** A wrapped 2D point $p'$ and an integer index $c$ identifying the repetition sector [S1, unofficial].

**Data Structures:**
- $p$: `vec2` (float x, float y).
- $p'$: `vec2` (float x, float y).
- $c$: `int` (sector index).
- $N$: `int` (repetitions).

**Steps:**
1. Compute the angular step size: $\theta = 2\pi / N$ [S1, unofficial].
2. Compute the raw angle and radius: $\alpha = \operatorname{atan2}(y, x)$, $\rho = \sqrt{x^2 + y^2}$ [S1, unofficial].
3. Shift the angle to center the modulo window: $\alpha_{shift} = \alpha + \theta/2$ [S1, unofficial].
4. Determine the sector index: $k = \lfloor \alpha_{shift} / \theta \rfloor$ [S1, unofficial].
5. Wrap the angle into the primary sector: $\alpha' = (\alpha_{shift} \pmod \theta) - \theta/2$ [S1, unofficial].
   - Note: The modulo operation must return a result in $[0, \theta)$ [S1, unofficial].
6. Reconstruct the point: $p' = (\cos \alpha', \sin \alpha') \cdot \rho$ [S1, unofficial].
7. Fix the sector index for symmetry: If $|k| \ge N/2$, set $k = |k|$ [S1, unofficial].
8. Return $p'$ and $k$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def polar_modulo(p: tuple[float, float], N: int) -> tuple[tuple[float, float], int]:
    PI = 3.141592653589793
    angle = 2.0 * PI / N
    a = math.atan2(p[1], p[0]) + angle / 2.0
    r = math.sqrt(p[0]**2 + p[1]**2)
    c = math.floor(a / angle)
    # GLSL-style mod: a - b * floor(a / b), returns [0, b)
    a = a - angle * math.floor(a / angle) - angle / 2.0
    p_prime = (math.cos(a) * r, math.sin(a) * r)
    if abs(c) >= N / 2:
        c = abs(c)
    return p_prime, c
```

**Parameters:**
- $N$: Number of repetitions. Not given in the sources.

**Edge Cases:**
- **Origin:** If $p = (0,0)$, $\rho=0$ and $\alpha$ is undefined. The result is $p'=(0,0)$ regardless of angle [G].
- **Precision:** Floating point errors in `atan2` or `mod` can cause discontinuities at the branch cut. Centering the modulo window ($\theta/2$ shift) helps align the discontinuity with the negative x-axis or specific symmetry axes [S1, unofficial].

## Subtopics

### Tentacle Wave with Phase Lag
A method to animate tentacles using polar repetition and phase offsets to create a trailing, unforced drift effect [S2, unofficial]. Each strand has a unique phase offset, and the wave arrives later further down the strand [S2, unofficial]. It is chosen when organic, non-rigid motion is required for radial patterns [S2, unofficial].

**Definition:** Extends Polar Modulo by adding time-dependent displacement and per-strand phase offsets [S2, unofficial].

**Steps:**
1. Place tentacles using Polar Modulo to determine base positions [S2, unofficial].
2. Assign each strand a phase offset $\phi$ derived from a hash function: $\phi = \operatorname{hash11}(\text{idx})$ [S2, unofficial].
3. Calculate wave displacement using depth-dependent amplitude and phase lag [S2, unofficial].

**Formula:**
The tentacle position is calculated as:
$$ \text{ring} = R\,(0.98 - 0.06\,d_n) + (0.05 + 0.78\,d_n^{2})\,\sin(1.55\,d - 2t + \phi) $$
where $R$ is the bell radius, $d_n$ is normalized distance along the strand, $d$ is distance, $t$ is time, and $\phi$ is the phase offset [S2, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Polar Modulo is a domain transformation technique used in procedural geometry and volume rendering to create radial symmetry [S1, unofficial]. It is implemented in GLSL-based rendering projects [S1, S2, unofficial].

## Sources
[S1] tdhooper/glsl-marching-cubes, https://github.com/tdhooper/glsl-marching-cubes, fetched 2026-10-26 (unofficial)
[S2] el-bakkali/manim-glsl-jellyfish, https://github.com/el-bakkali/manim-glsl-jellyfish, fetched 2026-10-26 (unofficial)
