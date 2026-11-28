# Frustum Culling for Octree Nodes

Also called: AABB-in-frustum test, Positive-vertex test

## Overview
Frustum culling for octree nodes is a hierarchical visibility test that discards entire octree subtrees if their axis-aligned bounding box (AABB) lies completely outside the view frustum [S1, unofficial]. It solves the problem of efficiently determining which parts of a sparse volumetric or point-cloud structure are visible, avoiding per-element processing for millions of points with a single bounding box test [S1, unofficial]. Unlike sibling methods that may integrate occlusion or frame-rate control, this method focuses purely on geometric containment within the six clipping planes, serving as the primary "existence" decision before traversal begins [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** An octree node defined by its axis-aligned bounding box (AABB) and the view frustum defined by six planes [S1, unofficial].
- **Output:** A boolean value indicating whether the node's bounding box intersects the frustum (True) or is completely outside (False) [S1, unofficial].

**Data Structures:**
- **Node:** Represented by two 3D vectors: `bmin` (minimum corner) and `bmax` (maximum corner) [S1, unofficial].
- **Planes:** A list of six plane definitions. Each plane is defined by the equation $ax + by + cz + d = 0$, where $(a, b, c)$ is the normal vector and $d$ is the distance from the origin [S1, unofficial].

**Algorithm Steps:**
1. For each of the six frustum planes, identify the corner of the node's bounding box that is farthest along the plane's normal vector [S1, unofficial].
2. The coordinates of this farthest corner $(p_x, p_y, p_z)$ are selected as follows: $p_x = \text{bmax}[0]$ if $a \ge 0$, else $\text{bmin}[0]$; $p_y = \text{bmax}[1]$ if $b \ge 0$, else $\text{bmin}[1]$; $p_z = \text{bmax}[2]$ if $c \ge 0$, else $\text{bmin}[2]$ [S1, unofficial].
3. Evaluate the plane equation for this corner: $val = a \cdot p_x + b \cdot p_y + c \cdot p_z + d$ [S1, unofficial].
4. If $val < 0$ for any of the six planes, the entire bounding box is outside the frustum relative to that plane; return False (cull the node) [S1, unofficial].
5. If the condition is not met for all six planes, the bounding box intersects the frustum; return True [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def aabb_in_frustum(node, planes):
    """True if the node's box is at least partly inside the frustum (positive-vertex test)."""
    bmin, bmax = node["min"], node["max"]
    for a, b, c, d in planes:
        px = bmax[0] if a >= 0 else bmin[0]
        py = bmax[1] if b >= 0 else bmin[1]
        pz = bmax[2] if c >= 0 else bmin[2]
        if a * px + b * py + c * pz + d < 0:      # most-positive corner still outside
            return False
    return True
```

**Parameters:**
- **Planes:** The six coefficients $(a, b, c, d)$ for the view frustum planes [S1, unofficial].
- **Node Bounds:** `bmin` and `bmax` define the spatial extent of the octree node [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Conservatism:** The test is conservative; it never hides a point that should be shown, ensuring correctness at the cost of potentially keeping some nodes that are mostly outside but have a corner inside [S1, unofficial].
- **Precision:** Standard floating-point arithmetic is sufficient, but care should be taken with near-zero plane distances or degenerate boxes [G].

## Operating characteristics
- **Time Complexity:** $O(1)$ per node, specifically costing six comparisons (one per frustum plane) to decide the existence of the entire subtree [S1, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space, requiring only local variables for the corner selection and plane evaluation [S1, unofficial].
- **Strengths:** Extremely fast hierarchical culling; discards millions of points with a single box test; simple implementation [S1, unofficial].
- **Limitations:** Conservative nature may lead to over-culling (keeping nodes that are mostly outside); does not account for occlusion by other geometry [S1, unofficial].
- **When to Use:** As a first-pass visibility filter for sparse volumes or point clouds where hierarchical structure is available [S1, unofficial].

## Background
Frustum culling for octree nodes originates from the need to efficiently render large-scale point clouds and sparse volumes [S1, unofficial]. The technique leverages the hierarchical nature of octrees to perform coarse-grained visibility tests [S1, unofficial]. It is a fundamental component in point cloud level-of-detail systems and out-core rendering pipelines [S1, unofficial]. The method is often referred to as the "positive-vertex test" because it tests the most positive vertex relative to each plane normal [S1, unofficial].

## Sources
[S1] Point Cloud Level of Detail: Octrees and Out-Core ..., https://learngeodata.eu/point-cloud-level-of-detail-octree/, fetched 2026-10-28 (unofficial)
