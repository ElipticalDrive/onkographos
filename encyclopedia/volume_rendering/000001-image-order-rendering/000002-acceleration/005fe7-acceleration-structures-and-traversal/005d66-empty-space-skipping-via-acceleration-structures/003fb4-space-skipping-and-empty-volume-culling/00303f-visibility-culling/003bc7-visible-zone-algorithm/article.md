# Visible Zone Algorithm

Also called: ComputeVisibilityMask, Hall-Holt and Rusinkiewicz Visible Zone Algorithm, Visibility Mask Computation

## Overview
The Visible Zone Algorithm is a conservative visibility computation method that determines which primitives are potentially visible from a set of viewpoints by projecting them into screen space and retaining only those within a specified depth thickness [S2, unofficial]. It sets itself apart from approximate visibility methods by guaranteeing correct visibility with less than one pixel of screen-space deviation [S2, unofficial].

## How it works
The algorithm computes a binary visibility matrix $O$ of size $N \times K$, where $N$ is the number of input points and $K$ is the number of viewpoints. It operates by projecting all points from each viewpoint, grouping them by pixel, sorting by depth, and marking the top $\tau$ points as visible [S1].

**Inputs:**
- $P$: Set of $N$ points (e.g., Gaussian centroids or voxel centers) [S1].
- $V$: Set of $K$ viewpoints [S1].
- $\tau$: Thickness parameter, defining the number of visible layers per pixel [S1].

**Outputs:**
- $O$: $N \times K$ visibility matrix, where $O[i, j] = 1$ if point $i$ is visible from viewpoint $j$, else $0$ [S1].

**Data Structures:**
- $M$: A dictionary mapping pixel coordinates $(u, v)$ to a list of points projected to that pixel [S1].
- $M^*$: A set of points identified as visible for the current viewpoint [S1].

**Steps:**
1. Initialize $O$ with zeros [S1].
2. For each viewpoint $j$ from $1$ to $K$:
   a. Let $c$ be the current viewpoint [S1].
   b. Retrieve extrinsic matrix $d$ and intrinsic matrix $K_{intr}$ for $c$ [S1].
   c. Initialize empty map $M$ [S1].
   d. For each point $i$ from $1$ to $N$:
      i. Let $p$ be point $P[i]$ [S1].
      ii. Compute 2D projection $x_c = (u, v)$ using forward projection with $d$ and $K_{intr}$ [S1].
      iii. Add $p$ to the list $M_{u,v}$ associated with pixel $(u, v)$ in $M$ [S1].
   e. Initialize empty set $M^*$ [S1].
   f. For each list $M_{u,v}$ in $M$:
      i. Sort $M_{u,v}$ by depth $z$ (ascending) [S1].
      ii. For $i$ from $1$ to $\tau$: add $M_{u,v}[i]$ to $M^*$ [S1].
   g. For each point $i$ from $1$ to $N$:
      i. If $P[i]$ is in $M^*$, set $O[i, j] = 1$ [S1].
3. Return $O$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_visibility_mask(P, V, tau):
    """
    Compute visibility mask for points P from viewpoints V with thickness tau.
    P: List of N points, each with 3D coordinates (x, y, z).
    V: List of K viewpoints, each with extrinsic (d) and intrinsic (K_intr) matrices.
    tau: Integer, number of visible layers per pixel.
    Returns: N x K matrix O.
    """
    N = len(P)
    K = len(V)
    O = [[0] * K for _ in range(N)]

    for j in range(K):
        c = V[j]
        d = c.extrinsic_matrix
        K_intr = c.intrinsic_matrix
        M = {}

        # Project all points
        for i in range(N):
            p = P[i]
            u, v = forward_projection(p, d, K_intr)
            if (u, v) not in M:
                M[(u, v)] = []
            M[(u, v)].append(p)

        M_star = set()

        # Determine visible points per pixel
        for point_list in M.values():
            # Sort by depth z
            point_list.sort(key=lambda pt: pt.z)
            # Pick first tau points (bounds to available points if tau exceeds list length)
            for k in range(min(tau, len(point_list))):
                M_star.add(point_list[k])

        # Update visibility matrix
        for i in range(N):
            if P[i] in M_star:
                O[i][j] = 1

    return O

def forward_projection(p, d, K_intr):
    """
    Project 3D point p using extrinsic d and intrinsic K_intr.
    Returns (u, v) pixel coordinates.
    """
    # Transform to camera coordinates
    p_cam = transform_point(p, d)
    # Perspective divide and project to pixel coordinates
    x, y, z = p_cam.x, p_cam.y, p_cam.z
    if z <= 0:
        return None
    u = K_intr.fx * (x / z) + K_intr.cx
    v = K_intr.fy * (y / z) + K_intr.cy
    return int(u), int(v)

def transform_point(p, d):
    """
    Apply 4x4 extrinsic matrix d to 3D point p.
    Returns transformed Point object.
    """
    wx = d[0][0]*p.x + d[0][1]*p.y + d[0][2]*p.z + d[0][3]
    wy = d[1][0]*p.x + d[1][1]*p.y + d[1][2]*p.z + d[1][3]
    wz = d[2][0]*p.x + d[2][1]*p.y + d[2][2]*p.z + d[2][3]
    return Point(wx, wy, wz)
```

**Parameters:**
- $\tau$: Thickness. Defines the number of visible layers per pixel [S1].

**Edge Cases and Pitfalls:**
- If $\tau$ exceeds the number of points projected to a pixel, the selection loop bounds to the available points [G].
- Precision in projection and sorting affects depth ordering and which points are considered "in front" [G].
- The algorithm assumes points are distinct; overlapping points at the same depth may require deterministic tie-breaking logic [G].

## Subtopics

### [Approximate Visibility Algorithm](13128-approximate-visibility-algorithm.md)
A method to compute approximate visibility for line drawings from volume data by reducing the number of vertices checked for occlusion, trading accuracy for performance. It sets itself apart from the exact visibility algorithm [Visible Zone Algorithm](9086-visible-zone-algorithm.md) by sacrificing worst-case correctness for a predictable speedup factor, making it suitable for large datasets where interactive frame rates are prioritized over perfect occlusion handling.

## Operating characteristics
The algorithm achieves real-time frame rates for 2D and 2.5D datasets [S2, unofficial]. The precomputation cost can be high; for a model with 13 million triangles, the accuracy guarantee costs 128 hours of precomputation on a cluster of 16 PCs [S2, unofficial]. It is suitable for scenarios where conservative visibility is critical and precomputation time is acceptable [S2, unofficial]. It is less suitable for dynamic scenes where viewpoints change frequently, as the mask must be recomputed [G].

## Background
The Visible Zone Algorithm was developed by Hall-Holt and Rusinkiewicz [S2, unofficial]. It is designed for conservative visibility computation with incremental updates [S2, unofficial]. It is used to interactively render large datasets with less than one pixel of screen-space deviation [S2, unofficial].

## Sources
[S1] Visibility-Based Geometry Pruning of Neural Plenoptic Scene Representations, https://doi.org/10.1109/tmm.2025.3618548, fetched 2026-10-26
[S2] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
