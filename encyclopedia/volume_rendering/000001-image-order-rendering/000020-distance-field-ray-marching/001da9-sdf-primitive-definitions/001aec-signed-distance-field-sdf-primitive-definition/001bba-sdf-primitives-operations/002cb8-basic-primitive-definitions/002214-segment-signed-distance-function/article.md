# Segment Signed Distance Function

Also called: Segment SDF, sdSegment, Distance to line segment

## Overview
The Segment Signed Distance Function computes the Euclidean distance from a query point $p$ to the finite line segment defined by endpoints $a$ and $b$ [S1, S3, S4, unofficial]. It solves the problem of determining proximity to bounded linear geometry, distinguishing itself from infinite line distance methods by clamping the projection parameter to the segment bounds $[0, 1]$ [S1, S3, S4, unofficial]. This ensures the result is zero if the point lies on the segment and correctly handles proximity to the endpoints [S1, unofficial]. It builds on [Basic Primitive Definitions](5346-basic-primitive-definitions.md).

## How it works
**Inputs:**
*   $p$: Query point (vector, typically 2D or 3D).
*   $a$: First endpoint of the segment.
*   $b$: Second endpoint of the segment.

**Outputs:**
*   $d$: The signed distance. For a segment, the "signed" aspect is often implicit in the context of SDF primitives where $d < 0$ is inside (though a 1D segment has no interior volume in standard Euclidean space, SDF libraries often return the absolute distance or treat the segment as the boundary of a capsule/solid) [S3, unofficial]. However, the core geometric operation returns the non-negative Euclidean distance to the closest point on the segment. In the context of SDF primitives, this value is used directly or offset by a radius [S3, unofficial].

**Data Structures:**
*   Vectors $p, a, b$ are stored as standard floating-point arrays (e.g., `vec2` or `vec3`) [S2, S3, unofficial][S5].

**Steps:**
1.  Compute the segment vector: $ab = b - a$ [S1, S3, unofficial].
2.  Compute the vector from the first endpoint to the query point: $ap = p - a$ [S1, S3, unofficial].
3.  Compute the squared length of the segment: $len\_sq = ab \cdot ab$ [S1, S3, S4, unofficial].
4.  Handle degenerate segments: If $len\_sq$ is near zero (e.g., $< 10^{-12}$ or component-wise $< 10^{-8}$), the segment is effectively a point. The closest point is $a$ (or $b$) [S3, S4, unofficial]. Return the distance $\|p - a\|$ (or squared distance if the variant requires it) [S3, S4, unofficial].
5.  Compute the projection parameter $t$ (or $h$):
    $$ t = \frac{ap \cdot ab}{len\_sq} $$
    Clamp $t$ to the range $[0, 1]$:
    $$ t = \text{clamp}(t, 0.0, 1.0) $$
    [S1, S3, unofficial][S5].
6.  Compute the closest point on the segment:
    $$ closest = a + ab \cdot t $$
    [S1, S4, unofficial][S5].
7.  Compute the distance vector:
    $$ dist\_vec = p - closest $$
    [S1, S3, unofficial].
8.  Return the distance:
    *   For squared distance: $d = dist\_vec \cdot dist\_vec$ [S1, unofficial].
    *   For Euclidean distance: $d = \|dist\_vec\| = \sqrt{dist\_vec \cdot dist\_vec}$ [S3, S4, unofficial].

**Formula (Normalized Variant):**
An alternative formulation normalizes the segment direction first [S2, unofficial]:
1.  $v = \frac{b - a}{\|b - a\|}$
2.  $l = \text{clamp}(v \cdot (p - a), 0, \|b - a\|)$
3.  $d = \|p - a - l \cdot v\|$

**Parameters:**
*   Epsilon for degenerate checks: $10^{-12}$ [S3, unofficial] or $10^{-8}$ [S4, unofficial].
*   Clamp range: $[0, 1]$ for parameter $t$ [S1, S3, unofficial][S5].

**Edge Cases and Pitfalls:**
*   **Degenerate Segment:** If $a = b$, the denominator in the projection calculation becomes zero. This must be checked explicitly to avoid division by zero [S3, S4, unofficial].
*   **Precision:** Use squared distances for comparisons when possible to avoid square roots, but return the Euclidean distance for the final SDF value [S1, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sdSegment(p, a, b):
    """
    Computes the Euclidean distance from point p to the segment [a, b].
    
    Args:
        p: Query point (vec2 or vec3)
        a: First endpoint (vec2 or vec3)
        b: Second endpoint (vec2 or vec3)
        
    Returns:
        float: Euclidean distance from p to the closest point on segment [a, b]
    """
    # 1. Segment vector
    ab = b - a
    
    # 2. Squared length of segment
    len_sq = dot(ab, ab)
    
    # 3. Handle degenerate segment (point)
    if len_sq < 1e-12:
        return length(p - a)
    
    # 4. Projection parameter
    ap = p - a
    t = dot(ap, ab) / len_sq
    
    # 5. Clamp to segment bounds
    t = max(0.0, min(1.0, t))
    
    # 6. Closest point
    closest = a + ab * t
    
    # 7. Distance
    return length(p - closest)
```

## Subtopics

### Point on Segment Check
Checks if a query point $Q$ lies on the line segment between endpoints $P_i$ and $P_j$ using distance comparison with epsilon tolerance [S4, unofficial].
*   **Definition:** Returns true if $Q$ is on the segment, false otherwise [S4, unofficial].
*   **Method:** Calculate distances $dis1 = \|P_i - Q\|$, $dis2 = \|P_j - Q\|$, and $dis3 = \|P_i - P_j\|$ [S4, unofficial]. If $dis3 + \epsilon < dis1 + dis2$, return false; otherwise, return true [S4, unofficial].
*   **Parameters:** $\epsilon = 10^{-8}$ [S4, unofficial].
*   **When to choose:** Use when exact geometric containment is required rather than proximity [S4, unofficial].

### Point-to-Segment Distance
Computes the closest point on a line segment $[v_0, v_1]$ to a query point $p$ [S5].
*   **Definition:** Returns the 3D coordinate of the closest point on the segment [S5].
*   **Method:** The result lies on the closed segment by clamping the parameter $t$ to $[0, 1]$ [S5].
*   **Parameters:** Supports any `Vec3` type with arithmetic and `dot()` operations [S5].
*   **When to choose:** Use when the closest point coordinate is needed for further geometric operations (e.g., normal calculation, ray intersection) rather than just the distance value [S5].

### Capsule Signed Distance Function — 2D Capsule SDF
Computes the signed distance to a 2D capsule (stadium shape) defined by endpoints $a$ and $b$ with radius $r$ [S3, unofficial].
*   **Definition:** The SDF of a capsule is the segment SDF minus the radius [S3, unofficial].
*   **Formula:** $d = \text{segment}(p, a, b) - r$ [S3, unofficial].
*   **Code:** `pub fn capsule2d(p: Vec2, a: Vec2, b: Vec2, r: f32) -> f32 { segment(p, a, b) - r }` [S3, unofficial].
*   **When to choose:** Use for rendering or collision detection of thick lines or capsules in 2D [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ constant time. Requires a few vector subtractions, dot products, and a square root [S1, S3, S4, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary space [S1, S3, unofficial].
*   **Strengths:** Simple, robust, and fast. Handles degenerate cases (points) gracefully [S3, S4, unofficial].
*   **Limitations:** Only applies to linear segments. Not suitable for complex curves without tessellation [S1, unofficial].
*   **When to use:** Ray marching for thin objects, collision detection for linear features, or as a building block for composite shapes like capsules [S2, S3, unofficial].
*   **When not to use:** When the geometry is an infinite line (use infinite line SDF) or a curve (use spline SDF) [S1, S3, unofficial].

## Background
The segment SDF is a fundamental primitive in Signed Distance Field rendering, popularized by Inigo Quilez's work on distance functions for ray marching [S2, S3, unofficial]. It is widely used in real-time graphics for rendering thin structures and as a component in more complex SDFs like capsules and cylinders [S2, S3, unofficial]. The algorithm is derived from basic vector projection and clamping logic [S1, S4, unofficial].

## Sources
[S1] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
[S2] erbenpeter/sdf, https://github.com/erbenpeter/sdf, fetched 2026-10-26 (unofficial)
[S3] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-07 (unofficial)
[S4] jacklv123/Volumetric-photon-mapping, https://github.com/jacklv123/Volumetric-photon-mapping, fetched 2026-10-07 (unofficial)
[S5] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
