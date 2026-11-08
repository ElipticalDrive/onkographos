# Box with Edge Rounding SDF
Also called: Rounded Box SDF, SDF Primitive: Box with Edge Rounding, sdBoxRounded

## Overview
Computes the signed distance to an axis-aligned box with rounded edges and corners, defined by half-extents $b$ and edge radius $e$ [S1, unofficial]. It extends the basic box distance by replacing sharp corners with spherical fillets, allowing for smooth geometric transitions in ray marching [S1, unofficial]. It differs from the [Unsigned Box Distance Function](3750-unsigned-box-distance-function.md) by providing signed interior/exterior information and from the [Rectangle Signed Distance Function](4031-rectangle-signed-distance-function.md) by supporting 3D volumes with uniform edge rounding rather than planar rectangles [G].

## How it works
**Inputs:**
- $p$: Query point (vec3) [S1, unofficial].
- $b$: Half-extents of the box (vec3) [S1, unofficial].
- $e$: Edge/corner rounding radius (float) [S1, unofficial].

**Outputs:**
- Signed distance value (float). Negative inside, positive outside, zero on surface [S1, unofficial].

**Data Structures:**
- Standard 3D vector types for $p$ and $b$. Scalar float for $e$ [G].

**Steps:**
1. Compute $d = |p| - b$ [S1, unofficial].
2. Compute $q = \max(d, 0)$ [S1, unofficial].
3. Compute $m = \max(d_x, \max(d_y, d_z))$ [S1, unofficial].
4. Return $\|q\| + \min(m, 0) - e$ [S1, unofficial].

**Formula:**
The signed distance is calculated by projecting the query point onto the box's exterior, measuring the distance to the closest face/edge/corner, and subtracting the rounding radius [S1, unofficial].
$$ \text{sd} = \| \max(|p| - b, 0) \| + \min(\max(|p| - b)_x, \max(|p| - b)_y, \max(|p| - b)_z) - e $$
where $| \cdot |$ is component-wise absolute value, $\max(\cdot)$ applies the maximum operator to each component or across components, $\| \cdot \|$ is the Euclidean length, and $e$ is the rounding radius [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sd_box_rounded(p, b, e):
    # p: vec3, b: vec3, e: float
    d = abs(p) - b
    q = max(d, 0.0)
    m = max(d.x, max(d.y, d.z))
    return length(q) + min(m, 0.0) - e
```

**Parameters:**
- $e$: Edge radius. Must satisfy $0 \le e \le \min(b_x, b_y, b_z)$ [S1, unofficial].

**Edge Cases:**
- $e = 0$: Reduces to the standard [Signed Box Distance Function](1092-signed-box-distance-function.md) [S1, unofficial].
- $e = \min(b)$: The box becomes a sphere with radius $\min(b)$ if $b_x=b_y=b_z$, or a capsule-like shape otherwise [G].
- Precision issues near the boundaries between face, edge, and corner regions [S1, unofficial].

## Subtopics

### Rounded Box2D SDF
Signed distance to a 2D rounded box with corner radius $r$ [S2, unofficial]. Formula: $\text{box2d}(p, b - r) - r$ [S2, unofficial]. Implementation: `pub fn rounded_box2d(p: Vec2, b: Vec2, r: f32) -> f32 { box2d(p, b - Vec2::splat(r)) - r }` [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per query [G].
- **Memory Complexity:** $O(1)$ [G].
- **Strengths:** Smooth geometry, efficient ray marching, no need for mesh tessellation [G].
- **Limitations:** Rounding radius must be uniform [G].
- **When to use:** When smooth, rounded boxes are required in ray marching [G].
- **When not to use:** When sharp corners are needed (use standard box) or when non-uniform rounding is required [G].

## Background
Originated from Inigo Quilez's work on SDFs [S1, unofficial].

## Sources
[S1] erbenpeter/sdf, https://github.com/erbenpeter/sdf, fetched 2026-10-26 (unofficial)
[S2] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-27 (unofficial)
