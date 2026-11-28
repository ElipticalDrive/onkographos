# Triangle Wave Replication Function

Also called: Triangle Wave Replication, g(t) = 1 + (2/pi) arcsin

## Overview
The Triangle Wave Replication Function is a mathematical tool used in procedural volume rendering and geometric modeling to create $C^1$-continuous cellular structures. It solves the continuity issues inherent in standard sawtooth replication functions, which can produce $C^0$-discontinuous functions or geometric artifacts on cell faces when the unit cell lacks full connectivity. It sets itself apart from its parent, the Sawtooth Replication Function, by ensuring smooth first-derivative continuity, provided the unit cell geometry is symmetric with respect to planes parallel to the cell faces and passing through their centers.

## How it works
The function maps a scalar input $t$ to a periodic triangle wave output $g(t)$ with a period of 2 and an amplitude range of $[0, 2]$. This mapping replaces the linear ramp of the sawtooth function with a triangular shape, eliminating the sharp corners that cause discontinuities in the derivative.

**Inputs and Outputs:**
- Input: $t \in \mathbb{R}$, a scalar coordinate value (typically normalized or scaled by a period parameter $a$).
- Output: $g(t) \in [0, 2]$, the replicated value.

**Data Structures:**
No complex data structures are required. The function is stateless and computed per-vertex or per-voxel.

**Formula:**
The exact formula for the triangle wave replication function is [S1]:

$$
g(t) = 1 + \frac{2}{\pi} \arcsin\left[\sin\left(\pi\left(t - \frac{1}{2}\right)\right)\right]
$$

**Parameters:**
- $t$: The input coordinate. In the context of cellular structures, this is often the coordinate value divided by the cell size $a$, i.e., $t = x/a$.
- The period is implicitly 2 due to the $\pi$ scaling inside the sine function.

**Steps:**
1. Shift the input $t$ by $-0.5$.
2. Multiply the shifted value by $\pi$.
3. Compute the sine of the result.
4. Compute the arcsine of the sine result.
5. Scale the arcsine result by $2/\pi$.
6. Add 1 to the result.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def triangle_wave_replication(t: float) -> float:
    """
    Computes the triangle wave replication function g(t).
    Returns a value in [0, 2].
    """
    # Step 1-3: sin(pi * (t - 0.5))
    inner = math.sin(math.pi * (t - 0.5))
    
    # Step 4: arcsin(inner)
    # Note: inner should be in [-1, 1]. Floating point errors might push it slightly outside.
    # Clamp to [-1, 1] to prevent NaNs [G].
    inner = max(-1.0, min(1.0, inner))
    
    arcsin_val = math.asin(inner)
    
    # Step 5-6: 1 + (2/pi) * arcsin_val
    result = 1.0 + (2.0 / math.pi) * arcsin_val
    
    return result
```

**Edge Cases and Pitfalls:**
- **Symmetry Requirement:** This function only yields $C^1$-continuous results if the unit cell geometry is symmetric with respect to planes parallel to the unit cell faces and passing through their center [S1]. If this symmetry is not present, the continuity benefits are lost.
- **Precision:** The `asin` function is sensitive to inputs outside $[-1, 1]$. Due to floating-point arithmetic, `sin` might return values slightly outside this range (e.g., $1.0000000000000002$), which would result in `NaN`. Clamping is necessary [G].

## Subtopics

### Fourier Sawtooth Replication
A variant that uses a Fourier series to approximate a sawtooth wave, allowing for $C^1$ and higher continuity but introducing artifacts. The formula is $g(t) = \frac{1}{2} - \frac{1}{\pi} \sum_{k=1}^{n} \frac{\sin(2\pi k a t)}{k}$, where $n$ is the number of harmonics [S1]. Using this method adds artifacts (projection-type copies of the unit cell geometry) to the neighborhood of cell faces, which is inappropriate in most cases [S1]. Even 50 harmonics are not enough for a good approximation of the sawtooth function [S1]. While Fourier series allows $C^1$ continuity, it requires many harmonics and introduces artifacts, whereas the standard triangle wave is simpler and artifact-free for symmetric cells [S1].

## Operating characteristics
- **Time Complexity:** $O(1)$ per evaluation. It involves basic trigonometric operations.
- **Memory Complexity:** $O(1)$. No auxiliary storage is needed.
- **Strengths:** Provides $C^1$ continuity for symmetric unit cells, avoiding the geometric discontinuities and surface artifacts associated with standard sawtooth replication when full connectivity is absent.
- **Limitations:** Requires the unit cell geometry to be symmetric with respect to the cell center planes. If this condition is not met, the continuity guarantee does not hold.
- **When to use:** When modeling cellular microstructures with symmetric unit cells and smooth blending operations are required.
- **When not to use:** When the unit cell lacks the required symmetry, or when computational cost of trigonometric functions is prohibitive (though usually negligible compared to other rendering steps).

## Background
The Triangle Wave Replication Function was introduced in the context of procedural function-based modeling of volumetric microstructures. It addresses the limitations of regular lattice models and standard cellular structures regarding geometric continuity. The method is detailed in the paper "Procedural function-based modelling of volumetric microstructures" [S1]. The approach builds on the idea that while regular lattices have better analytical properties, they are limited in geometry. Cellular structures offer more geometric flexibility but suffer from continuity issues at cell faces. The triangle wave was selected as an alternative to the sawtooth wave to achieve $C^1$ continuity for symmetric unit cells [S1].

## Sources
[S1] Procedural function-based modelling of volumetric microstructures, https://doi.org/10.1016/j.gmod.2011.03.001, fetched 2026-10-26
