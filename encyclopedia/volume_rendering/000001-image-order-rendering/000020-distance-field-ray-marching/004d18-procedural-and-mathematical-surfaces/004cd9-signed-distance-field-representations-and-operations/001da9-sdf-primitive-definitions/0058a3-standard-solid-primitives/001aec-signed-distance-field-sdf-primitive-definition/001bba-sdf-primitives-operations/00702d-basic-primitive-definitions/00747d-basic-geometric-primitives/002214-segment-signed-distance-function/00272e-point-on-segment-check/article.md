# Point on Segment Check

Also called: onSegment, point-on-segment test

## Overview
The Point on Segment Check determines whether a query point $Q$ lies exactly on the line segment defined by endpoints $P_i$ and $P_j$. It solves the geometric containment problem for 1D segments embedded in 2D or 3D space. It is distinguished from [Line Segment Distance](3058-line-segment-distance.md) and [Point-to-Segment Distance](4205-point-to-segment-distance.md) by returning a binary boolean result rather than a scalar distance value, serving as a primitive predicate within the [Segment Signed Distance Function](2679-segment-signed-distance-function.md).

## How it works
**Inputs:**
- $P_i$: First endpoint of the segment (vector).
- $P_j$: Second endpoint of the segment (vector).
- $Q$: Query point (vector).
- $\epsilon$: Floating-point tolerance for equality comparison.

**Outputs:**
- `true` if $Q$ lies on the segment $P_i P_j$; `false` otherwise.

**Data Structures:**
- Vectors $P_i, P_j, Q$ are typically 3D Cartesian coordinates $(x, y, z)$.
- Distance is Euclidean: $d(A, B) = \sqrt{(A_x-B_x)^2 + (A_y-B_y)^2 + (A_z-B_z)^2}$.

**Steps:**
1. Compute the distance from $P_i$ to $Q$: $d_1 = \text{distance}(P_i, Q)$.
2. Compute the distance from $P_j$ to $Q$: $d_2 = \text{distance}(P_j, Q)$.
3. Compute the length of the segment $P_i P_j$: $d_3 = \text{distance}(P_i, P_j)$.
4. Compare the sum of the sub-segments to the total length using the tolerance $\epsilon$.
   - If $d_3 + \epsilon < d_1 + d_2$, return `false`.
   - Otherwise, return `true`.

**Formulas:**
$$ d_1 = \| Q - P_i \| $$
$$ d_2 = \| Q - P_j \| $$
$$ d_3 = \| P_j - P_i \| $$
Condition for containment:
$$ d_1 + d_2 \le d_3 + \epsilon $$

**Parameters:**
- $\epsilon$: Epsilon for floating-point comparison. Typical value is $10^{-8}$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Degenerate Segments:** If $P_i = P_j$, then $d_3 = 0$. The check returns `true` only if $Q$ is also equal to $P_i$ (within $\epsilon$).
- **Floating Point Precision:** Direct equality checks fail due to precision errors. The distance-sum method is preferred over ratio-based methods (checking if $Q$ divides $P_i P_j$ in a specific ratio) because ratio methods fail when the segment aligns with a primary axis, causing division by zero [S2, unofficial].
- **Rounding Tolerance:** Some implementations use rounding (e.g., `Math.Round(dist2 + dist3, 3) == Math.Round(dist1, 3)`) instead of additive epsilon, though additive epsilon is more standard in continuous geometry [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def on_segment(Pi, Pj, Q, eps=1e-8):
    """
    Check if point Q lies on the segment between Pi and Pj.
    
    Args:
        Pi: Tuple or list of 3 floats (x, y, z)
        Pj: Tuple or list of 3 floats (x, y, z)
        Q:  Tuple or list of 3 floats (x, y, z)
        eps: Floating point tolerance (default 1e-8)
        
    Returns:
        bool: True if Q is on the segment, False otherwise
    """
    def distance(A, B):
        return math.sqrt((A[0]-B[0])**2 + (A[1]-B[1])**2 + (A[2]-B[2])**2)
    
    dis1 = distance(Pi, Q)
    dis2 = distance(Pj, Q)
    dis3 = distance(Pi, Pj)
    
    if dis3 + eps < dis1 + dis2:
        return False
    return True
```

## Operating characteristics
- **Time Complexity:** $O(1)$ constant time. Requires 3 distance calculations (each involving 3 subtractions, 3 multiplications, 2 additions, and 1 square root) and 2 additions/comparisons.
- **Memory Complexity:** $O(1)$ auxiliary space.
- **Strengths:** Numerically stable; avoids division by zero errors associated with parametric/ratio checks; simple to implement.
- **Limitations:** Sensitive to the choice of $\epsilon$ in high-precision or large-scale coordinate systems. The square root operations are computationally more expensive than squared-distance checks, but necessary here because the triangle inequality ($d_1 + d_2 \ge d_3$) holds for Euclidean distances, not squared distances.

## Background
The method is a standard geometric primitive used in computational geometry and computer graphics. It appears in implementations of volumetric photon mapping and voxel game engines as a helper for collision detection or point-in-segment validation [S1, S2, unofficial]. The distance-sum approach is cited as a robust alternative to ratio-based methods which suffer from axis-alignment singularities [S2, unofficial].

## Sources
[S1] jacklv123/Volumetric-photon-mapping, https://github.com/jacklv123/Volumetric-photon-mapping, fetched 2026-10-27 (unofficial)
[S2] TerryLyons16/MonoGame_VoxelGameEngine, https://github.com/TerryLyons16/MonoGame_VoxelGameEngine, fetched 2026-10-27 (unofficial)
