# Smooth Intersection

Also called: Round Intersection, S-Intersection, Smooth SDF Operations

## Overview
Smooth Intersection is a Constructive SDF operator that blends the boolean intersection of two signed distance fields into a rounded feature controlled by a blending radius $r$ (or smoothing parameter $k$) [S1, unofficial][S2, unofficial][S3, unofficial]. It is distinguished from generic smooth minimums like [SDF Smooth Union](1817-sdf-smooth-union.md) by its specific geometric construction aimed at preserving distance field properties [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
- $a$: Distance from the first SDF.
- $b$: Distance from the second SDF.

**Outputs:**
- $d$: The resulting blended distance value.

**Data Structures:**
None required; scalar operations [S3, unofficial].

**Steps:**
1. Compute $m = \max(a, b)$ [S3, unofficial].
2. Evaluate the condition $-a < r$ and $-b < r$ [S3, unofficial].
3. If the condition is true, compute $f = \max(m, -(r - \sqrt{(r+a)^2 + (r+b)^2}))$ [S3, unofficial].
4. If the condition is false, set $f = m$ [S3, unofficial].
5. Return $f$ [S3, unofficial].

**Formulas:**
The sources provide two distinct mathematical formulations for the operation [S1, unofficial][S3, unofficial][S4, unofficial].

1. **Geometric/Vector Formulation (S-Intersection):**
   This method calculates the distance to a rounded corner formed by the intersection [S3, unofficial].
   $$ f = \begin{cases} \max\left(m, -\left(r - \sqrt{(r+a)^2 + (r+b)^2}\right)\right) & \text{if } -a < r \land -b < r \\ m & \text{otherwise} \end{cases} $$
   [S3, unofficial]

2. **Polynomial/Blending Formulation:**
   This method uses a polynomial blend based on the difference between distances [S4, unofficial].
   $$ h = \max(k - |a - b|, 0.0) $$
   $$ d = \min(a, b) - \frac{h^2}{4k} $$
   [S4, unofficial]

   *Alternative Polynomial Formulation:*
   Source [S1, unofficial] and [S2, unofficial] describe a similar polynomial approach using a mixing factor $h$:
   $$ h = \text{clamp}\left(0.5 - 0.5 \frac{b - a}{k}, 0.0, 1.0\right) $$
   $$ d = \text{mix}(b, a, h) + k \cdot h \cdot (1.0 - h) $$
   where $\text{mix}(x, y, t) = x(1-t) + yt$ [S1, unofficial][S2, unofficial].

**Parameters:**
- $r$: Blending radius or smoothing parameter [S2, unofficial].
- Typical values: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** Not given in the sources.
- **Lipschitz Continuity:** Not given in the sources.
- **Usage Guidelines:** Not given in the sources.

**Pseudocode:**
The following implements the Geometric/Vector Formulation (S-Intersection) [S3, unofficial].
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def smooth_intersection(a: float, b: float, r: float) -> float:
    m = max(a, b)
    if (-a < r) and (-b < r):
        dist_to_corner = math.sqrt((r + a)**2 + (r + b)**2)
        return max(m, -(r - dist_to_corner))
    else:
        return m
```

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.

## Background
Smooth Intersection operators are documented in unofficial repositories related to sphere tracing and differentiable rendering [S1, unofficial][S2, unofficial][S3, unofficial]. The specific formulations are attributed to these community implementations [S1, unofficial][S2, unofficial][S4, unofficial].

## Sources
[S1] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] skmhrk1209/Sphere-Tracing-PyTorch3D, https://github.com/skmhrk1209/Sphere-Tracing-PyTorch3D, fetched 2026-10-26 (unofficial)
[S3] tdhooper/glsl-marching-cubes, https://github.com/tdhooper/glsl-marching-cubes, fetched 2026-10-26 (unofficial)
[S4] JamesRicher/TerminalSignedDistanceFieldRenderer, https://github.com/JamesRicher/TerminalSignedDistanceFieldRenderer, fetched 2026-10-27 (unofficial)
