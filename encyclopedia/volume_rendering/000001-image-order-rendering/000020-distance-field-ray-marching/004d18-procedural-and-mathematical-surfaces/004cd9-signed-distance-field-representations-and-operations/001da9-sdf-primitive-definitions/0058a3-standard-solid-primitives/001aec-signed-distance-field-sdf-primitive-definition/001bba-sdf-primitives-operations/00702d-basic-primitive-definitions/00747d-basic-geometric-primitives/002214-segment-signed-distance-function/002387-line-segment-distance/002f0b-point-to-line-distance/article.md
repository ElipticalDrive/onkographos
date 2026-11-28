# Point-to-Line Distance
Also called: DistToLineSq, Distance to Line Squared, Point-to-Line Vector

## Overview
Computes the vector from a point to the closest point on an infinite line, or the squared Euclidean distance to that line [S1, unofficial][S2, unofficial]. It serves as the geometric core for [Line Segment Distance](3058-line-segment-distance.md), which clamps this result to segment bounds [G]. Unlike segment distance, this method assumes an infinite extent, making it suitable for directional constraints or unbounded ray approximations [G].

## How it works
**Inputs:**
- `p`: The query point (3D vector).
- `o`: A point on the line (origin, 3D vector).
- `d`: The direction vector of the line (3D vector).

**Outputs:**
- `qp`: The vector from the closest point on the line to `p` (3D vector).
- `distSq`: The squared Euclidean distance (scalar).

**Data Structures:**
Standard 3D floating-point vectors. No complex packing required.

**Steps:**
1. Compute the projection scale factor `scale_factor` representing the position along the direction vector `d` closest to `p`:
   $$ \text{scale\_factor} = \frac{\mathbf{op} \cdot \mathbf{d}}{\mathbf{d} \cdot \mathbf{d}} $$
   where $\mathbf{op} = \mathbf{p} - \mathbf{o}$ [S1, unofficial].
2. Compute the closest point `q` on the line:
   $$ \mathbf{q} = \mathbf{o} + \text{scale\_factor} \times \mathbf{d} $$ [S1, unofficial].
3. Compute the residual vector `qp` from the closest point to the query point:
   $$ \mathbf{qp} = \mathbf{p} - \mathbf{q} $$ [S1, unofficial].
4. Compute the squared distance:
   $$ \text{distSq} = \|\mathbf{qp}\|^2 $$ [S2, unofficial].

**Formula:**
The squared distance is computed via orthogonal projection subtraction [S1, unofficial]:
$$ \text{distSq} = \| (\mathbf{p} - \mathbf{o}) - \frac{(\mathbf{p} - \mathbf{o}) \cdot \mathbf{d}}{\mathbf{d} \cdot \mathbf{d}} \mathbf{d} \|^2 $$
where $\mathbf{op} = \mathbf{p} - \mathbf{o}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_to_line_distance(p, o, d):
    # Vector from line origin to point
    op = p - o
    
    # Dot products
    dot_op_d = dot(op, d)
    dot_d_d = dot(d, d)
    
    # Guard against zero-length direction vector
    if dot_d_d == 0.0:
        # Direction is zero; closest point is the origin itself
        qp = p - o
        dist_sq = dot(qp, qp)
        return qp, dist_sq
        
    # Projection scale factor
    scale_factor = dot_op_d / dot_d_d
    
    # Closest point on the infinite line
    q = o + scale_factor * d
    
    # Vector from closest point to original point
    qp = p - q
    
    # Squared Euclidean distance
    dist_sq = dot(qp, qp)
    
    return qp, dist_sq
```

**Parameters:**
- `d` must be non-zero. If `d` is zero, the line is undefined [G]. Implementation should handle `dot(d, d) == 0` to avoid division by zero [G].

**Edge Cases:**
- **Zero Direction:** If $\mathbf{d} = \mathbf{0}$, the line collapses to a single point $\mathbf{o}$ [G]. The distance is $\|\mathbf{p} - \mathbf{o}\|$ [G].
- **Precision:** For very long lines or small distances, floating-point cancellation may occur in $\mathbf{p} - \mathbf{q}$ [G]. Using the projection formula directly on the normalized direction may improve stability if $\mathbf{d}$ is normalized beforehand [G].

## Operating characteristics
- **Time Complexity:** $O(1)$ constant time [G]. Requires a few dot products and vector additions [G].
- **Memory Complexity:** $O(1)$ auxiliary space [G].
- **Strengths:** Extremely fast; no branching required if direction is guaranteed non-zero [G].
- **Limitations:** Does not account for segment endpoints [G]. Using this for a segment without clamping yields incorrect distances near the ends [G].

## Background
This method is a fundamental linear algebra operation used in computer graphics, physics engines, and geometric modeling [G].

## Sources
[S1] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
[S2] Real-time BC6H compression on GPU, https://github.com/knarkowicz/GPURealTimeBC6H, fetched 2026-10-27 (unofficial)
