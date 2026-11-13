# Point-to-Line Distance
Also called: DistToLineSq, Distance to Line Squared, Point-to-Line Vector

## Overview
Computes the vector from a point to the closest point on an infinite line, or the squared Euclidean distance to that line. It serves as the geometric core for [Line Segment Distance](3058-line-segment-distance.md), which clamps this result to segment bounds. Unlike segment distance, this method assumes an infinite extent, making it suitable for directional constraints or unbounded ray approximations.

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
   where $\mathbf{op} = \mathbf{p} - \mathbf{o}$.
2. Compute the closest point `q` on the line:
   $$ \mathbf{q} = \mathbf{o} + \text{scale\_factor} \times \mathbf{d} $$
3. Compute the residual vector `qp` from the closest point to the query point:
   $$ \mathbf{qp} = \mathbf{p} - \mathbf{q} $$
4. Compute the squared distance:
   $$ \text{distSq} = \|\mathbf{qp}\|^2 $$

**Formula:**
The squared distance can also be expressed directly via projection subtraction [S2, unofficial]:
$$ \text{distSq} = \| (\mathbf{p} - \mathbf{o}) - ((\mathbf{p} - \mathbf{o}) \cdot \hat{\mathbf{d}}) \hat{\mathbf{d}} \|^2 $$
where $\hat{\mathbf{d}}$ is the normalized direction vector. Note that [S1, unofficial] uses the unnormalized direction in the projection step, requiring division by $\|\mathbf{d}\|^2$.

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
- `d` must be non-zero. If `d` is zero, the line is undefined. [G] Implementation should handle `dot(d, d) == 0` to avoid division by zero.

**Edge Cases:**
- **Zero Direction:** If $\mathbf{d} = \mathbf{0}$, the line is a single point $\mathbf{o}$. The distance is $\|\mathbf{p} - \mathbf{o}\|$.
- **Precision:** For very long lines or small distances, floating-point cancellation may occur in $\mathbf{p} - \mathbf{q}$. Using the projection formula directly on the normalized direction may improve stability if $\mathbf{d}$ is normalized beforehand.

## Operating characteristics
- **Time Complexity:** $O(1)$ constant time. Requires a few dot products and vector additions.
- **Memory Complexity:** $O(1)$ auxiliary space.
- **Strengths:** Extremely fast; no branching required if direction is guaranteed non-zero.
- **Limitations:** Does not account for segment endpoints. Using this for a segment without clamping yields incorrect distances near the ends.

## Background
This method is a fundamental linear algebra operation used in computer graphics, physics engines, and geometric modeling. It appears in contexts such as Gaussian Splatting rasterization [S1, unofficial] and texture compression bounding box refinements [S2, unofficial].

## Sources
[S1] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
[S2] Real-time BC6H compression on GPU, https://github.com/knarkowicz/GPURealTimeBC6H, fetched 2026-10-27 (unofficial)
