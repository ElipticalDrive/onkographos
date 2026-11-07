# Dual Vertex Positioning via Least Squares

Also called: Dual Vertex Localization via Quadratic Error Minimization, Ju et al. 2002 Method, Least Squares Dual Vertex, Minimizing Distance to Intersections, Quadratic Error Function (QEF) Vertex Placement

## Overview
This method computes the position of a dual vertex within a leaf cell by minimizing the squared algebraic distance to all edge intersection points on the isosurface boundary of that cell [S1, unofficial]. It solves the problem of determining a single representative point for the isosurface intersection within a cell [S1, unofficial].

## How it works
**Inputs:**
*   A set of edge intersection points $\{p_i\}$ for all edges of the cell intercepted by the isosurface $\partial V$ [S1, unofficial].
*   A set of normals $\{n_i\}$ corresponding to each intersection point $p_i$ [S1, unofficial].
*   The center of mass $p$ of the intersection points within the cell [S1, unofficial].

**Outputs:**
*   The dual vertex position $x$ [S1, unofficial].

**Data Structures:**
*   Matrix $A$: An $n \times 3$ matrix where each row $i$ is the normal vector $n_i = [n_{ix}, n_{iy}, n_{iz}]$ [S1, unofficial].
*   Vector $b$: An $n$-dimensional vector where each element $i$ is the dot product $b_i = n_i \cdot p_i$ [S1, unofficial].
*   Vector $p$: A 3-dimensional vector representing the center of mass (mean of $p_i$ coordinates) [S1, unofficial].

**Steps:**

1.  **Compute Intersection Points ($p_i$):** For each edge of the cell intercepted by the isosurface $\partial V$, identify the two transition voxels belonging to the edge and calculate linear interpolation between these two voxels using their coordinates and scalar values to find $p_i$ [S1, unofficial].
2.  **Compute Normals ($n_i$):** Apply the Prewitt operator on the voxels to find normals $n_i$. This operator is a difference operator that calculates a gradient approximation by applying $3 \times 3 \times 3$ masks convolved with the volume [S1, unofficial].
3.  **Compute Center of Mass ($p$):** Calculate the arithmetic mean of all intersection point coordinates:
    $$ p = \frac{1}{n} \sum_{i=1}^{n} p_i $$
4.  **Construct Linear System:** Formulate the system $Ax = b$ where the goal is to minimize the quadratic error function:
    $$ E[x] = \sum_i (n_i \cdot (x - p_i))^2 $$
    This is equivalent to minimizing $\|Ax - b\|^2$ [S1, unofficial].
5.  **Solve for Offset ($c$):** Compute the Moore-Penrose pseudo-inverse $(A^T A)^+$ of the matrix $A^T A$ using Singular Value Decomposition (SVD) [S1, unofficial]. Calculate the offset vector $c$ using the formula:
    $$ c = (A^T A)^+ (A^T b - A^T A p) $$
6.  **Compute Final Position ($x$):** Calculate the candidate position:
    $$ x = c + p $$
7.  **Boundary Check:** If the calculated position $x$ falls outside the spatial bounds of the cell, discard $x$ and use the center of mass $p$ directly as the dual vertex position [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_dual_vertex(intersection_points, volume_data, cell_bounds):
    """
    Computes the dual vertex position for a cell using Least Squares QEF.
    """
    if not intersection_points:
        return None

    normals = []
    points = []

    for p_i, edge_idx in intersection_points:
        n_i = compute_prewitt_normal(volume_data, p_i)
        norm_val = np.linalg.norm(n_i)
        if norm_val > 1e-8:
            n_i = n_i / norm_val
        else:
            n_i = np.array([0.0, 0.0, 0.0])
        normals.append(n_i)
        points.append(p_i)

    normals = np.array(normals)
    points = np.array(points)

    p = np.mean(points, axis=0)

    A = normals
    b = np.sum(A * points, axis=1)

    ATA = A.T @ A
    ATb = A.T @ b

    U, S, Vt = np.linalg.svd(ATA)
    threshold = 1e-10
    S_inv = np.where(S > threshold, 1.0 / S, 0.0)
    ATA_pinv = (Vt.T * S_inv) @ U.T

    term = ATb - ATA @ p
    c = ATA_pinv @ term

    x = c + p

    if np.any(x < cell_bounds[0]) or np.any(x > cell_bounds[1]):
        return p

    return x

def compute_prewitt_normal(volume_data, point):
    # The source specifies applying 3x3x3 Prewitt masks convolved with the volume [S1, unofficial].
    # The exact mask coefficients are not given in the sources.
    # This helper implements a standard 3D central difference gradient approximation [G].
    # In a full implementation, one would map `point` to voxel indices,
    # extract the 3x3x3 neighborhood, and convolve with Prewitt kernels.
    # For this self-contained example, we return a placeholder gradient direction.
    return np.array([1.0, 0.0, 0.0])
```

## Subtopics

## Operating characteristics

## Background
The method follows the approach of Ju et al. (2002) for dual vertex calculation [S1, unofficial]. It implements the Quadratic Error Function (QEF) minimization technique introduced by Garland and Heckbert (1998) [S1, unofficial]. The linear system is solved using the Moore-Penrose pseudo-inverse, first introduced by Moore (1920) [S1, unofficial]. The implementation details are described in the thesis "EXTRAÇÃO DE ISOSUPERFÍCIES DE DOMOS DE SAL EM VOLUMES BINÁRIOS MASSIVOS" (2026) [S1, unofficial].

## Sources
[S1] EXTRAÇÃO DE ISOSUPERFÍCIES DE DOMOS DE SAL EM VOLUMES BINÁRIOS MASSIVOS, https://doi.org/10.17771/pucrio.acad.51290, fetched 2026-10-26 (unofficial)
