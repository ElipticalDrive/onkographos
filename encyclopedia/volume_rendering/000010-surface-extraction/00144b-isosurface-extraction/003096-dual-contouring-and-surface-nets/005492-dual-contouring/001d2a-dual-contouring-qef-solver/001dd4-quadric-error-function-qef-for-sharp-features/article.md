# Quadric Error Function (QEF) for Sharp Features

Also called: QEF for Sharp Features, QEF Solver, Quadric Error Function Solver, QEF Solver using SVD

## Overview
Quadric Error Function (QEF) for Sharp Features is a vertex relocation technique used in Dual Contouring to position a single vertex within an octree cell such that it accurately represents sharp features (edges and corners) of the isosurface [S1, unofficial]. It solves the problem of standard Dual Contouring producing smoothed or inaccurate vertices at high-curvature regions by minimizing the sum of squared distances from the vertex to the tangent planes defined by isosurface intersection points [S1, unofficial]. It sets itself apart from its siblings, such as [Brute-Force QEF Solver](4403-brute-force-qef-solver.md) and [QEF Solver Initialization](4722-qef-solver-initialization.md), by explicitly constructing "supporting planes" in addition to tangent planes to stabilize the minimization process and prevent numerical instabilities inherent in standard Quadric Error Metric (QEM) approaches [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A set of $N$ intersection point-normal pairs $\{(p_i, n_i)\}$ for $i=0 \dots N-1$, where $p_i$ is the position where the isosurface intersects an edge of the cell, and $n_i$ is the gradient normal at that point [S1, unofficial].
*   **Output:** A 3D vertex position $v$ that minimizes the combined quadric error function [S1, unofficial].

**Data Structures**
The method accumulates constraints into a symmetric data structure [S2, unofficial]. Based on [S2, unofficial], the `QefData` structure holds the following fields:
*   `ata_00`, `ata_01`, `ata_02`, `ata_11`, `ata_12`, `ata_22`: Elements of the symmetric $3 \times 3$ matrix $A^T A$ (or $ATA$) [S2, unofficial].
*   `atb_x`, `atb_y`, `atb_z`: Elements of the vector $A^T b$ (or $ATB$) [S2, unofficial].
*   `btb`: Scalar value representing the accumulated squared dot products [S2, unofficial].
*   `massPoint_x`, `massPoint_y`, `massPoint_z`: The accumulated sum of intersection points (used to compute the centroid) [S2, unofficial].
*   `numPoints`: Integer count of intersection points $N$ [S2, unofficial].

**Algorithm Steps**

1.  **Compute Mass Point and Normal:**
    Calculate the mass point $p_{mass}$ as the arithmetic mean of all intersection points $p_i$:
    $$ p_{mass} = \frac{1}{N} \sum_{i=0}^{N-1} p_i $$
    Calculate the mass normal $n_{mass}$ as the average of all intersection normals $n_i$, normalized:
    $$ n_{mass} = \frac{\sum_{i=0}^{N-1} n_i}{\left\| \sum_{i=0}^{N-1} n_i \right\|} $$
    [S1, unofficial]

2.  **Construct Tangent Plane Quadrics ($Q_I$):**
    For each intersection point $p_i$ with normal $n_i$, define a tangent plane equation $n_i^T (v - p_i) = 0$ [S3, unofficial]. This corresponds to the standard QEM formulation [S3, unofficial].
    The quadric for a single plane $n^T v + d = 0$ (where $d = -n^T p_i$) is defined by:
    $$ A = n n^T $$
    $$ b = d n $$
    $$ c = d^2 $$
    Accumulate these into the global $ATA$, $ATB$, and $btb$ matrices/vectors [S2, unofficial]. Specifically, for each $i$:
    *   Normalize $n_i$.
    *   Compute $dot_i = n_i \cdot p_i$.
    *   Update $ATA \leftarrow ATA + n_i n_i^T$ [S2, unofficial].
    *   Update $ATB \leftarrow ATB + dot_i n_i$ [S2, unofficial].
    *   Update $btb \leftarrow btb + dot_i^2$ [S2, unofficial].
    [S1, unofficial]

3.  **Construct Supporting Plane Quadrics ($Q_S$):**
    To stabilize the solution for sharp features, construct supporting planes for each intersection point [S1, unofficial].
    For each $i$:
    *   Compute the edge vector $e$:
        $$ e = \frac{p_{mass} - p_i}{\| p_{mass} - p_i \|} $$
        *Pitfall:* If $p_{mass} = p_i$, $e$ is zero [S1, unofficial].
    *   Compute the intermediate normal $n_e$:
        $$ n_e = \frac{n_{mass} + n_i}{\| n_{mass} + n_i \|} $$
    *   Compute the supporting plane normal $n_s$:
        $$ n_s = \frac{n_e \times e}{\| n_e \times e \|} $$
        *Pitfall:* If $e$ is parallel to $n_e$, the cross product is zero and the supporting plane normal is undefined [S1, unofficial].
    *   The supporting plane passes through $p_{mass}$ with normal $n_s$ [S1, unofficial]. The plane equation is $n_s^T (v - p_{mass}) = 0$.
    *   Let $dot_s = n_s \cdot p_{mass}$.
    *   Accumulate the supporting plane quadric into $ATA$, $ATB$, and $btb$ using the same logic as Step 2, but scaled by a factor $a$ [S1, unofficial].

4.  **Combine Quadrics:**
    The final error function $E(v)$ is the sum of the tangent quadrics $Q_I$ and the supporting quadrics $Q_S$, weighted by factor $a$:
    $$ Q = Q_I + a Q_S $$
    The parameter $a$ controls feature sharpness [S1, unofficial]. A typical value is $0.5$ [S1, unofficial]. Small values produce sharper features but may introduce instability; large values smooth the feature more [S1, unofficial].

5.  **Solve the Linear System:**
    The problem is to minimize $E(v) = v^T (ATA) v - 2 (ATB)^T v + btb$ [S2, unofficial]. This is equivalent to solving the linear system:
    $$ (ATA) v = ATB $$
    Since $ATA$ may be rank-deficient or ill-conditioned (especially for edges where $N < 3$), use Singular Value Decomposition (SVD) [S2, unofficial] [S3, unofficial].
    
    *Centering:* To improve numerical stability, shift the coordinate system to the mass point [S2, unofficial].
    Let $v = x + p_{mass}$.
    The system becomes:
    $$ (ATA) x = ATB - (ATA) p_{mass} $$
    Let $b' = ATB - (ATA) p_{mass}$ [S2, unofficial].
    
    *SVD Solve:*
    Compute the SVD of $ATA = U D V^T$. Since $ATA$ is symmetric, $U=V$ [S2, unofficial].
    Compute the pseudoinverse of the diagonal matrix $D$, denoted $D^+$, where singular values $\sigma_j < \text{pinv\_tol}$ are treated as zero [S2, unofficial].
    $$ x = V D^+ U^T b' $$
    [S2, unofficial]
    
    *Final Position:*
    $$ v = x + p_{mass} $$
    [S2, unofficial]

**Parameters**
*   $a$: Scaling factor for supporting planes. Typical value: $0.5$ [S1, unofficial].
*   `svd_tol`: Tolerance for SVD convergence/singular value thresholding. Not specified in sources.
*   `svd_sweeps`: Maximum number of Givens rotation sweeps for the SVD algorithm. Not specified in sources.
*   `pinv_tol`: Tolerance for pseudoinverse computation. Singular values below this are treated as zero [S2, unofficial]. Not specified in sources.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def solve_qef_sharp_features(intersection_points, intersection_normals, a=0.5, pinv_tol=1e-8):
    """
    Solves the QEF for sharp features using supporting planes and SVD.
    
    Args:
        intersection_points: List of 3D vectors (p_i).
        intersection_normals: List of 3D vectors (n_i).
        a: Weight for supporting planes (default 0.5).
        pinv_tol: Tolerance for pseudoinverse computation.
        
    Returns:
        Vertex position v.
    """
    n = len(intersection_points)
    if n == 0:
        raise ValueError("No intersection points provided")

    # Step 1: Compute Mass Point and Mass Normal
    mass_point = [0.0, 0.0, 0.0]
    mass_normal = [0.0, 0.0, 0.0]
    
    for i in range(n):
        mass_point[0] += intersection_points[i][0]
        mass_point[1] += intersection_points[i][1]
        mass_point[2] += intersection_points[i][2]
        mass_normal[0] += intersection_normals[i][0]
        mass_normal[1] += intersection_normals[i][1]
        mass_normal[2] += intersection_normals[i][2]
        
    mass_point[0] /= n; mass_point[1] /= n; mass_point[2] /= n
    norm_mass_normal = math.sqrt(mass_normal[0]**2 + mass_normal[1]**2 + mass_normal[2]**2)
    if norm_mass_normal == 0:
        return mass_point
    mass_normal[0] /= norm_mass_normal; mass_normal[1] /= norm_mass_normal; mass_normal[2] /= norm_mass_normal

    # Initialize ATA, ATB, btb
    ata = [[0.0]*3 for _ in range(3)]
    atb = [0.0, 0.0, 0.0]
    btb = 0.0

    # Step 2 & 3: Accumulate Tangent and Supporting Plane Quadrics
    for i in range(n):
        p_i = intersection_points[i]
        n_i = intersection_normals[i]
        
        norm_n_i = math.sqrt(n_i[0]**2 + n_i[1]**2 + n_i[2]**2)
        if norm_n_i == 0:
            continue
        n_i = [n_i[0]/norm_n_i, n_i[1]/norm_n_i, n_i[2]/norm_n_i]

        # --- Tangent Plane Quadric (Q_I) ---
        dot_i = n_i[0]*p_i[0] + n_i[1]*p_i[1] + n_i[2]*p_i[2]
        
        # A = n_i * n_i^T
        ata[0][0] += n_i[0]*n_i[0]; ata[0][1] += n_i[0]*n_i[1]; ata[0][2] += n_i[0]*n_i[2]
        ata[1][0] += n_i[1]*n_i[0]; ata[1][1] += n_i[1]*n_i[1]; ata[1][2] += n_i[1]*n_i[2]
        ata[2][0] += n_i[2]*n_i[0]; ata[2][1] += n_i[2]*n_i[1]; ata[2][2] += n_i[2]*n_i[2]
        
        # b = dot_i * n_i
        atb[0] += dot_i * n_i[0]; atb[1] += dot_i * n_i[1]; atb[2] += dot_i * n_i[2]
        
        # c += dot_i^2
        btb += dot_i ** 2

        # --- Supporting Plane Quadric (Q_S) ---
        diff = [mass_point[0]-p_i[0], mass_point[1]-p_i[1], mass_point[2]-p_i[2]]
        norm_diff = math.sqrt(diff[0]**2 + diff[1]**2 + diff[2]**2)
        
        if norm_diff > 1e-8:
            e = [diff[0]/norm_diff, diff[1]/norm_diff, diff[2]/norm_diff]
            
            n_e_raw = [mass_normal[0]+n_i[0], mass_normal[1]+n_i[1], mass_normal[2]+n_i[2]]
            norm_n_e = math.sqrt(n_e_raw[0]**2 + n_e_raw[1]**2 + n_e_raw[2]**2)
            
            if norm_n_e > 1e-8:
                n_e = [n_e_raw[0]/norm_n_e, n_e_raw[1]/norm_n_e, n_e_raw[2]/norm_n_e]
                
                # n_s = n_e x e
                n_s = [n_e[1]*e[2]-n_e[2]*e[1], n_e[2]*e[0]-n_e[0]*e[2], n_e[0]*e[1]-n_e[1]*e[0]]
                norm_n_s = math.sqrt(n_s[0]**2 + n_s[1]**2 + n_s[2]**2)
                
                if norm_n_s > 1e-8:
                    n_s = [n_s[0]/norm_n_s, n_s[1]/norm_n_s, n_s[2]/norm_n_s]
                    
                    dot_s = n_s[0]*mass_point[0] + n_s[1]*mass_point[1] + n_s[2]*mass_point[2]
                    
                    # Accumulate with weight a
                    ata[0][0] += a * n_s[0]*n_s[0]; ata[0][1] += a * n_s[0]*n_s[1]; ata[0][2] += a * n_s[0]*n_s[2]
                    ata[1][0] += a * n_s[1]*n_s[0]; ata[1][1] += a * n_s[1]*n_s[1]; ata[1][2] += a * n_s[1]*n_s[2]
                    ata[2][0] += a * n_s[2]*n_s[0]; ata[2][1] += a * n_s[2]*n_s[1]; ata[2][2] += a * n_s[2]*n_s[2]
                    
                    atb[0] += a * dot_s * n_s[0]; atb[1] += a * dot_s * n_s[1]; atb[2] += a * dot_s * n_s[2]
                    btb += a * dot_s ** 2

    # Step 5: Solve (ATA) v = ATB using SVD with centering
    tmp = [0.0, 0.0, 0.0]
    tmp[0] = ata[0][0]*mass_point[0] + ata[0][1]*mass_point[1] + ata[0][2]*mass_point[2]
    tmp[1] = ata[1][0]*mass_point[0] + ata[1][1]*mass_point[1] + ata[1][2]*mass_point[2]
    tmp[2] = ata[2][0]*mass_point[0] + ata[2][1]*mass_point[1] + ata[2][2]*mass_point[2]
    
    rhs = [atb[0]-tmp[0], atb[1]-tmp[1], atb[2]-tmp[2]]
    
    # SVD of ATA (symmetric)
    u, s, vh = np.linalg.svd(ata)
    
    # Pseudoinverse of singular values
    s_inv = [0.0]*len(s)
    for j in range(len(s)):
        if s[j] > pinv_tol:
            s_inv[j] = 1.0 / s[j]
        else:
            s_inv[j] = 0.0
            
    # x = Vh.T @ diag(s_inv) @ U.T @ rhs
    x = [0.0, 0.0, 0.0]
    for r in range(3):
        for c in range(3):
            x[r] += vh[r][c] * s_inv[c] * sum(u[c][k]*rhs[k] for k in range(3))
    
    v = [x[0]+mass_point[0], x[1]+mass_point[1], x[2]+mass_point[2]]
    return v
```

## Subtopics

### [QEF Boundary Constrained Vertex Placement](1740-qef-boundary-constrained-vertex-placement.md)
A fallback strategy for when the unconstrained QEF solution lies outside the voxel cell. It constrains the search space hierarchically (interior -> faces -> edges -> corners). It sets itself apart by handling boundary violations explicitly rather than relying solely on SVD minimization.

## Operating characteristics
*   **Time Complexity:** Dominated by the SVD of a $3 \times 3$ matrix, which is constant time $O(1)$ per cell [G]. Accumulating the quadrics is $O(N)$ where $N$ is the number of intersection points (typically 3 to 12) [S1, unofficial] [S2, unofficial].
*   **Memory Complexity:** $O(1)$ per cell for storing the accumulated $ATA$, $ATB$, and $btb$ values [S2, unofficial].
*   **Performance:** Computationally efficient compared to general-purpose QEF solvers or iterative methods [S1, unofficial].
*   **Strengths:** Preserves sharp features (edges and corners) better than standard Dual Contouring or Marching Cubes. Stabilizes the solution against numerical instabilities that occur with standard QEM when planes are nearly parallel or coincident [S1, unofficial].
*   **Limitations:** Requires careful handling of degenerate cases (zero vectors, parallel normals) to avoid division by zero or invalid cross products [S1, unofficial]. The choice of parameter $a$ affects sharpness vs. stability trade-off [S1, unofficial].
*   **When to use:** When extracting isosurfaces with sharp features using Dual Contouring and vertex accuracy at corners/edges is critical [S1, unofficial].
*   **When not to use:** For smooth surfaces where feature preservation is not desired, or when computational resources are extremely limited and approximate smoothing is acceptable [G].

## Background
The Quadric Error Metric (QEM) was introduced by Garland and Heckbert in 1997 [S1, unofficial]. It was originally designed for mesh simplification but was adapted for isosurface extraction in Dual Contouring by Ju et al. (2002) [S1, unofficial]. The specific variation using supporting planes to stabilize sharp feature extraction was detailed in later works on Dual Contouring for terrain and complex surface modeling [S1, unofficial].

## Sources
[S1] Generating Smooth High-Quality Isosurfaces for Interactive Modeling and Visualization of Complex Terrains, https://doi.org/10.2312/pe/vmv/vmv12/079-086, fetched 2026-10-26 (unofficial)
[S2] nickgildea/DualContouringSample, https://github.com/nickgildea/DualContouringSample, fetched 2026-10-26 (unofficial)
[S3] http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, fetched 2026-10-30 (unofficial)
