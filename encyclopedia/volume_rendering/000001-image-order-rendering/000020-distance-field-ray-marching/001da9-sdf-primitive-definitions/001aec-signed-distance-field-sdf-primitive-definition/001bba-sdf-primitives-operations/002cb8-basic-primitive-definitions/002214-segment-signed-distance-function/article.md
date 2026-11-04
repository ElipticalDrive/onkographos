# Segment Signed Distance Function

Also called: Segment SDF, sdSegment, Distance to line segment

## Overview
The Segment Signed Distance Function computes the Euclidean distance from a query point $p$ to the closest point on a line segment defined by endpoints $a$ and $b$ [S1, S3, S4, unofficial]. It serves as a fundamental primitive for constructing complex shapes in Signed Distance Field (SDF) rendering, allowing for the representation of thin structures, edges, and skeletal geometries [S2, unofficial]. Unlike infinite line primitives, this method explicitly handles the segment boundaries, ensuring the distance is zero only within the segment's extent [S5, unofficial]. It is distinct from [SDF Primitives](1731-sdf-primitives.md) in its specific focus on 1D linear geometry, and from [Elongation](2440-elongation.md) as it defines the base geometry rather than modifying an existing volume.

## How it works
**Inputs:**
- $p$: The query point (vector type, e.g., `vec2` or `vec3`) [S2, unofficial][S6].
- $a$: The first endpoint of the segment [S1, S3, unofficial].
- $b$: The second endpoint of the segment [S1, S3, unofficial].

**Outputs:**
- $d$: The Euclidean distance $\|p - \text{closest}\|$ [S1, S3, unofficial].

**Data Structures:**
- Vectors $a, b, p$ are stored as standard floating-point arrays (e.g., 2 or 3 components) [S2, unofficial][S6].
- Intermediate scalars $t$ (or $h, u$) represent the normalized projection parameter [S1, S3, S4, unofficial].

**Steps:**
1. Compute the segment vector: $ab = b - a$ [S1, unofficial].
2. Compute the vector from the start point to the query point: $ap = p - a$ [S1, unofficial].
3. Compute the squared length of the segment: $len\_sq = ab \cdot ab$ [S1, S3, unofficial].
4. Handle degenerate segments (zero length):
   - If $len\_sq < \epsilon$ (where $\epsilon$ is a small value like $10^{-12}$ or $10^{-8}$ depending on implementation precision), the segment is a point. Return $\|p - a\|$ [S3, S4, unofficial].
   - [G] In practice, use a tolerance appropriate for the coordinate scale to avoid division by zero.
5. Compute the projection parameter $t$ (or $h$):
   - Calculate the dot product: $dot = ap \cdot ab$ [S1, S3, unofficial].
   - Normalize by squared length: $t = dot / len\_sq$ [S1, S3, unofficial].
   - Clamp $t$ to the range $[0, 1]$ to restrict the closest point to the segment: $t = \text{clamp}(t, 0.0, 1.0)$ [S1, S3, unofficial].
   - [S4, unofficial] uses a slightly different formulation: $u = \frac{(p - a) \cdot (b - a)}{\|b - a\|^2}$, which is mathematically equivalent.
6. Compute the closest point on the segment: $closest = a + ab \cdot t$ [S1, unofficial].
   - [S4, unofficial] computes this as $begin + u \cdot (begin - end)$, noting that $begin - end = -(end - begin)$. If $a=begin$ and $b=end$, then $ab = end - begin$, so $a + u(ab) = begin + u(end - begin)$. The source [S4, unofficial] uses $begin - end$ in the numerator dot product but also in the final addition, effectively calculating $u$ relative to the vector $begin-end$. Let's align with the standard [S1/S3] approach for clarity: $closest = a + t(b-a)$.
7. Return the distance: $d = \|p - closest\|$ [S1, S3, unofficial].
   - [S1, unofficial] notes returning the squared distance in one variant, but the standard SDF returns the Euclidean distance [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sdSegment(p, a, b):
    """
    Computes the Euclidean distance from point p to the segment [a, b].
    p, a, b are vectors of the same dimension (2D or 3D).
    """
    # 1. Vector from a to b
    ab = b - a
    
    # 2. Vector from a to p
    ap = p - a
    
    # 3. Squared length of the segment
    len_sq = dot(ab, ab)
    
    # 4. Handle degenerate segment (point)
    # Use a small epsilon for floating point comparison
    epsilon = 1e-12
    if len_sq < epsilon:
        return length(ap)
    
    # 5. Project p onto the line, clamped to [0, 1]
    # t represents the fraction of the segment length
    t = dot(ap, ab) / len_sq
    t = clamp(t, 0.0, 1.0)
    
    # 6. Find the closest point on the segment
    closest = a + ab * t
    
    # 7. Return the Euclidean distance
    return length(p - closest)
```

**Parameters:**
- $\epsilon$: Epsilon for degenerate check. Common values are $10^{-12}$ [S3, unofficial] or $10^{-8}$ [S4, unofficial].
- $p, a, b$: Coordinates in the same space (2D or 3D) [S2, unofficial][S6].

**Edge Cases and Pitfalls:**
- **Degenerate Segments:** If $a = b$, the denominator $len\_sq$ becomes zero. This must be checked explicitly to avoid division by zero [S3, S4, unofficial].
- **Precision:** When $p$ is very close to the segment, floating-point errors in the dot product and division can cause slight inaccuracies. Using `clamp` ensures the result remains physically valid (distance $\ge 0$) [S1, S3, unofficial].
- **Dimensionality:** The algorithm works for any dimension $N$ as long as vector arithmetic is supported [S6].

## Subtopics

### Line Segment Distance
Computes the Euclidean distance from a point $p$ to a line segment defined by endpoints $l_0$ and $l_1$ [S5, unofficial]. This variant explicitly details the branch logic for determining which endpoint is closer when the projection falls outside the segment, rather than relying on a generic clamp function [S5, unofficial]. It is useful when implementing custom clamping logic or optimizing for specific hardware branches. Choose this when you need explicit control over the "closer to start" vs "closer to end" conditions [S5, unofficial].

### Point on Segment Check
Checks if a point $Q$ lies on the line segment between $P_i$ and $P_j$ using distance comparison with an epsilon tolerance [S4, unofficial]. It calculates distances $dis1$ ($P_i$ to $Q$), $dis2$ ($P_j$ to $Q$), and $dis3$ ($P_i$ to $P_j$). If $dis3 + \epsilon < dis1 + dis2$, it returns false; otherwise true [S4, unofficial]. This is a geometric predicate rather than a distance function, used for validation or collision detection [S4, unofficial]. Choose this when you need to verify collinearity and bounds rather than compute a distance value [S4, unofficial].

### Point-to-Segment Distance
Computes the closest point on a line segment $[v_0, v_1]$ to a query point $p$ [S6]. The result is guaranteed to lie on the closed segment by clamping the parameter $t$ to $[0, 1]$ [S6]. This is a helper function often used in conjunction with the SDF to retrieve the geometric closest point for shading or normal calculation [S6]. Choose this when the closest point location is needed, not just the distance [S6].

## Operating characteristics
- **Time Complexity:** $O(1)$ constant time. It involves a fixed number of vector subtractions, dot products, and a square root [S1, S3, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space. No large data structures are required [S1, S3, unofficial].
- **Performance:** Extremely fast, suitable for real-time ray marching in fragment shaders or CPU-based path tracing [S2, S4, unofficial].
- **Strengths:** Simple implementation, numerically stable with proper epsilon handling, works in any dimension [S3, unofficial][S6].
- **Limitations:** Does not handle infinite lines (use [SDF Domain Manipulation Operators](1730-sdf-domain-manipulation-operators.md) for line extensions) or thick segments (use [Elongation](2440-elongation.md) or capsules) [S5, unofficial].
- **When to use:** For rendering thin wires, edges, or skeletal structures in SDF scenes [S2, unofficial].
- **When not to use:** For infinite lines or when the segment thickness is significant [S5, unofficial].

## Background
The Segment SDF is a standard primitive in the field of Signed Distance Fields, popularized by Inigo Quilez's articles on distance functions [S3, unofficial]. It is widely used in ray marching techniques for rendering complex geometries composed of simple primitives [S2, unofficial]. The mathematical formulation relies on projecting a point onto a line segment and clamping the parameter to the segment's bounds [S1, S3, unofficial]. Implementations can be found in various graphics libraries and educational repositories [S1, S2, S3, S4, S5, unofficial][S6].

## Sources
[S1] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
[S2] erbenpeter/sdf, https://github.com/erbenpeter/sdf, fetched 2026-10-26 (unofficial)
[S3] bhubbard/sdf-math-rs, https://github.com/bhubbard/sdf-math-rs, fetched 2026-10-07 (unofficial)
[S4] jacklv123/Volumetric-photon-mapping, https://github.com/jacklv123/Volumetric-photon-mapping, fetched 2026-10-07 (unofficial)
[S5] Amory0709/geresim-spike, https://github.com/Amory0709/geresim-spike, fetched 2026-10-07 (unofficial)
[S6] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
