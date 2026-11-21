# Capsule Signed Distance Function

Also called: 3D Capsule SDF, Capsule Rendering via Distance Check, Capsule SDF, Capsule3D SDF, Signed Capsule, sdCapsule

## Overview
The Capsule Signed Distance Function computes the exact Euclidean distance from a query point in 3D space to the surface of a capsule, defined as a line segment with a constant radius [S3, unofficial]. It represents smooth, rounded tubular geometry for ray marching and volume rendering without polygonal approximation [G]. It is distinguished from the [Sphere SDF](1237-sphere-signed-distance-function.md) by its linear axis and from the [Box SDF](1092-signed-box-distance-function.md) by its cylindrical symmetry and hemispherical ends [G].

## How it works
**Inputs and Outputs:**
- Input: Query point $p \in \mathbb{R}^3$, capsule endpoints $a, b \in \mathbb{R}^3$, radius $r \in \mathbb{R}^+$ [S3, unofficial].
- Output: Signed distance $d \in \mathbb{R}$. $d < 0$ if inside, $d > 0$ if outside, $d = 0$ on surface [G].

**Data Structures:**
The evaluation requires only scalar and vector variables; no auxiliary data structures are needed [G].

**Parameters:**
- $p$: The point to test [S4, unofficial].
- $a, b$: The start and end points of the capsule's central axis [S3, unofficial].
- $r$: The radius of the capsule. Typical values depend on scene scale; [S4, unofficial] hardcodes $r=0.12$.

**Algorithm:**
The capsule is the Minkowski sum of a line segment $\overline{ab}$ and a sphere of radius $r$ [G]. The distance is the distance from $p$ to the segment $\overline{ab}$, minus $r$ [G].
1. Compute the vector representing the capsule axis: $ba = b - a$ [S3, unofficial].
2. Compute the vector from the start point to the query point: $pa = p - a$ [S3, unofficial].
3. Compute the squared length of the axis: $len\_sq = ba \cdot ba$ [S5, unofficial].
4. Handle degenerate case: If $len\_sq < \epsilon$ (where $\epsilon$ is a small positive number, e.g., $10^{-12}$), the capsule is a sphere centered at $a$. Return $length(pa) - r$ [G].
5. Compute the projection parameter $h$ of $pa$ onto $ba$, clamped to $[0, 1]$:
   $$ h = \text{clamp}\left( \frac{pa \cdot ba}{len\_sq}, 0.0, 1.0 \right) $$ [S3, unofficial].
   This $h$ identifies the closest point on the infinite line passing through $a$ and $b$ that lies within the segment [G].
6. Compute the closest point on the segment: $closest = a + ba \cdot h$ [S3, unofficial].
7. Compute the vector from the closest point to $p$: $diff = p - closest = pa - ba \cdot h$ [S3, unofficial].
8. Return the distance: $d = length(diff) - r$ [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sd_capsule(p: Vec3, a: Vec3, b: Vec3, r: float) -> float:
    ba = b - a
    pa = p - a
    len_sq = dot(ba, ba)
    # Guard for degenerate segment (not in sources)
    if len_sq < 1e-12:
        return length(pa) - r
    h = clamp(dot(pa, ba) / len_sq, 0.0, 1.0)
    closest = a + ba * h
    diff = p - closest
    return length(diff) - r
```

**Edge Cases and Pitfalls:**
- **Degenerate Segment:** If $a = b$, the division by $len\_sq$ causes NaN. A guard check for $len\_sq < \epsilon$ is required [G].
- **Precision:** For very long capsules, $len\_sq$ may overflow if not using double precision, though standard float32 handles typical scene scales [G].
- **Axis Alignment:** The formula is general for any orientation. An axis-aligned variant (e.g., Y-axis aligned) exists [S1, unofficial].

## Subtopics

### Axis-Aligned Capsule SDF
A specialized version where the capsule is aligned with a coordinate axis (e.g., Y-axis). It simplifies the projection step by only clamping the corresponding coordinate [S1, unofficial]. Choose this when rendering static, axis-aligned tubes for performance optimization. See [Axis-Aligned Capsule SDF](link).

### 2D Capsule SDF
The 2D equivalent, computing distance to a line segment with rounded ends in a plane. Useful for 2D volumetric rendering or sprite-based effects [S5, unofficial]. See [2D Capsule SDF](link).

## Operating characteristics
- **Time Complexity:** $O(1)$. The calculation involves a constant number of vector operations (subtraction, dot product, length) [G].
- **Memory Complexity:** $O(1)$. No auxiliary data structures are allocated [G].
- **Performance:** The primitive evaluation is extremely fast, making it suitable for real-time ray marching and volume rendering [G].
- **Strengths:** Exact distance, smooth normals (derivable via finite differences or analytical gradient), supports arbitrary orientation and position [G].
- **Limitations:** Cannot represent non-uniform thickness (tapered capsules require more complex functions) [G].

## Background
The Capsule SDF is a standard primitive in Signed Distance Field rendering, implemented in GLSL [S3, unofficial] and Rust [S5, unofficial]. It is widely used in ray marching demos, procedural geometry generation, and collision detection in real-time graphics [G]. The mathematical formulation relies on the property that the distance to a capsule is the distance to its central segment minus the radius [S3, unofficial].

## Sources
[S1] Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S2] Volumetric-display, https://github.com/quertzuiop/Volumetric-display, fetched 2026-10-26 (unofficial)
[S3] glsl-sdf-primitives, https://github.com/marklundin/glsl-sdf-primitives, fetched 2026-10-26 (unofficial)
[S4] sdf, https://github.com/erbenpeter/sdf, fetched 2026-10-26 (unofficial)
[S5] sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
