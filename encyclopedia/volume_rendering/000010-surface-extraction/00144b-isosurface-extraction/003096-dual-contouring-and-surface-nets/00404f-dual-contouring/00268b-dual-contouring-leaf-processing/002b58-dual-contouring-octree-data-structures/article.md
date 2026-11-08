# Dual Contouring Octree Data Structures

Also called: DC Octree Layout, Tao Ju DC Node Types

## Overview
The Dual Contouring implementation uses an octree with three node types to represent the implicit surface: `InternalNode` (empty space or non-leaf structure), `LeafNode` (voxel containing surface intersections with a QEF), and `PseudoLeafNode` (merged children that form a surface patch but are not at max depth) [S1, unofficial]. It differs from uniform grid approaches by using these distinct node types to handle varying levels of geometric detail, building on the leaf processing logic defined in [Dual Contouring Leaf Processing](3817-dual-contouring-leaf-processing.md) [S1, unofficial].

## How it works

### Inputs and Outputs
The method takes implicit function evaluations at voxel corners, producing vertex signs and, for intersected edges, intersection points and surface normals [S1, unofficial]. It outputs an octree structure where each node contains either empty space, a single vertex position (minimizer) derived from a QEF, or pointers to child nodes [S1, unofficial]. The final output is a set of vertices and connectivity information derived from these nodes [S1, unofficial].

### Data Structures
The implementation uses a base `OctreeNode` type extended by `QEFMixin` and specific node classes [S1, unofficial]. All coordinates are 3D floats.

#### QEFMixin
This base class stores the data required to fit a plane to the surface within a voxel [S1, unofficial]. It is inherited by both `LeafNode` and `PseudoLeafNode` [S1, unofficial].
**Fields:**
* `unsigned char signs`: A bitmask representing the sign (inside/outside) of the 8 voxel corners. Bit $i$ corresponds to vertex $i$ [S1, unofficial].
* `char height`: The depth of the node in the octree (0 for root, increasing for children) [S1, unofficial].
* `float mp[3]`: The QEF minimizer point (the generated vertex position) [S1, unofficial].
* `int index`: The index of this vertex in the final output PLY file [S1, unofficial].
* `float ata[6]`: The upper triangular part of the $A^T A$ matrix for the QEF [S1, unofficial]. Stored in row-major order for the upper triangle: `ata[0]` = $A^T A_{00}$, `ata[1]` = $A^T A_{01}$, `ata[2]` = $A^T A_{02}$, `ata[3]` = $A^T A_{11}$, `ata[4]` = $A^T A_{12}$, `ata[5]` = $A^T A_{22}$ [S1, unofficial].
* `float atb[3]`: The $A^T b$ vector. `atb[0]` = $A^T b_0$, `atb[1]` = $A^T b_1$, `atb[2]` = $A^T b_2$ [S1, unofficial].
* `float btb`: The scalar $b^T b$ term [S1, unofficial].

#### LeafNode
Represents a voxel at the maximum depth of the octree that contains surface intersections [S1, unofficial].
**Fields:**
* Inherits all fields from `OctreeNode` and `QEFMixin` [S1, unofficial].
* `float st[3]`: The starting corner (minimum coordinates) of the voxel [S1, unofficial].
* `float len`: The side length of the voxel [S1, unofficial].
* `int numint`: The number of edges intersected by the surface (0 to 12) [S1, unofficial].
* `float inters[12][3]`: The intersection points on the 12 edges [S1, unofficial].
* `float norms[12][3]`: The surface normals at the intersection points [S1, unofficial].

#### PseudoLeafNode
Represents a cluster of children that have been merged into a single surface patch but are not at the maximum leaf depth [S1, unofficial].
**Fields:**
* Inherits all fields from `OctreeNode` and `QEFMixin` [S1, unofficial].
* `OctreeNode * child[8]`: Pointers to the 8 children. In the context of contouring, if the children are fully merged, these pointers may be NULL or point to merged structures, but the structural slot exists [S1, unofficial].

#### InternalNode
Represents empty space or a non-leaf structure that does not contain surface intersections requiring a vertex [S1, unofficial]. It typically contains only the `child[8]` pointers [S1, unofficial].

### Edge and Face Mappings
The algorithm relies on fixed mappings between cube geometry and array indices [S1, unofficial].

**Vertex Indexing:**
Vertices are indexed 0–7. The standard ordering is:
* 0: (0,0,0), 1: (1,0,0), 2: (0,1,0), 3: (1,1,0)
* 4: (0,0,1), 5: (1,0,1), 6: (0,1,1), 7: (1,1,1) [S1, unofficial]

**Edge-to-Vertex Mapping:**
The 12 edges are mapped to pairs of vertex indices (0-7). Edge 0 connects vertices 0 and 4. Edge 1 connects 1 and 5, etc. [S1, unofficial]

**Face-to-Edge Mapping:**
The 6 faces are mapped to the 4 edges bounding them. Face 0 uses edges 4, 8, 5, 9. Face 1 uses 6, 10, 7, 11. Face 2 uses 0, 8, 1, 10. Face 3 uses 2, 9, 3, 11. Face 4 uses 0, 4, 2, 6. Face 5 uses 1, 5, 3, 7 [S1, unofficial].

**Child Pair to Face Direction:**
The 12 faces between 8 children are processed by `faceProcContour`. The mask `cellProcFaceMask[12][3]` lists child pairs and direction. E.g., pair (0,4) is direction 0. Pair (0,2) is direction 1. Pair (0,1) is direction 2 [S1, unofficial].

**Child Quadruplets to Edge Direction:**
The 6 internal edges shared by 4 children are processed by `edgeProcContour`. `cellProcEdgeMask[6][5]` lists the 4 child indices and direction. E.g., children {0,1,2,3} share edge direction 0 [S1, unofficial].

### QEF Assembly and Minimization
For a `LeafNode`, the QEF is assembled from the intersection points and normals [S1, unofficial].

1. **QEF Assembly:** For each intersection `i` from 0 to `numint-1`, accumulate the QEF matrices: `ata[0] += norm[0]*norm[0]`, `ata[1] += norm[0]*norm[1]`, `ata[2] += norm[0]*norm[2]`, `ata[3] += norm[1]*norm[1]`, `ata[4] += norm[1]*norm[2]`, `ata[5] += norm[2]*norm[2]` [S1, unofficial]. Accumulate `atb`: `atb[0] += norm[0]*pn`, `atb[1] += norm[1]*pn`, `atb[2] += norm[2]*pn` where `pn = p[0]*norm[0] + p[1]*norm[1] + p[2]*norm[2]` [S1, unofficial]. Accumulate `btb += pn*pn` [S1, unofficial]. Also sum points `pt` for averaging [S1, unofficial].
2. **Minimizer Calculation:** After accumulating QEF data, compute the average intersection point `pt` by dividing sums by `numint` [S1, unofficial]. Call `calcPoint(ata, atb, btb, pt, mp, box, mat)` to solve for the minimizer `mp` [S1, unofficial]. If `CLAMP` is defined and `mp` is outside the bounding box `[st, st+len]`, replace `mp` with the average point `pt` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def assemble_qef_and_solve(st, len_val, numint, inters, norms):
    """
    Assembles the QEF for a leaf node and solves for the minimizer.
    
    Args:
        st: float[3], start corner of the voxel.
        len_val: float, side length of the voxel.
        numint: int, number of intersections.
        inters: float[numint][3], intersection points.
        norms: float[numint][3], surface normals at intersections.
        
    Returns:
        float[3], the minimizer point mp.
    """
    # Initialize QEF matrices and vectors
    ata = [0.0] * 6
    atb = [0.0] * 3
    btb = 0.0
    pt_sum = [0.0, 0.0, 0.0]
    
    # Accumulate QEF data
    for i in range(numint):
        nx, ny, nz = norms[i]
        px, py, pz = inters[i]
        
        # Dot product p . n
        pn = px * nx + py * ny + pz * nz
        
        # Accumulate A^T A (upper triangular)
        ata[0] += nx * nx
        ata[1] += nx * ny
        ata[2] += nx * nz
        ata[3] += ny * ny
        ata[4] += ny * nz
        ata[5] += nz * nz
        
        # Accumulate A^T b
        atb[0] += nx * pn
        atb[1] += ny * pn
        atb[2] += nz * pn
        
        # Accumulate b^T b
        btb += pn * pn
        
        # Accumulate point sum for averaging
        pt_sum[0] += px
        pt_sum[1] += py
        pt_sum[2] += pz
        
    # Calculate average point
    if numint > 0:
        pt = [pt_sum[0] / numint, pt_sum[1] / numint, pt_sum[2] / numint]
    else:
        # Edge case: No intersections
        pt = [st[0] + len_val / 2.0, st[1] + len_val / 2.0, st[2] + len_val / 2.0]
        
    # Solve for minimizer
    # [G] Standard 3x3 linear solver implementation
    mp = solve_qef(ata, atb, pt, st, len_val)
    
    return mp

def solve_qef(ata, atb, pt, st, len_val):
    # Construct full 3x3 A^T A matrix from upper triangular ata
    A = [
        [ata[0], ata[1], ata[2]],
        [ata[1], ata[3], ata[4]],
        [ata[2], ata[4], ata[5]]
    ]
    # Solve A * mp = atb using Gaussian elimination with partial pivoting [G]
    mp = gaussian_solve_3x3(A, atb)
    
    # Clamping logic
    box_min = st
    box_max = [st[0] + len_val, st[1] + len_val, st[2] + len_val]
    if is_outside_box(mp, box_min, box_max):
        return pt
    return mp

def gaussian_solve_3x3(A, b):
    # Standard 3x3 linear solver [G]
    M = [A[0] + [b[0]], A[1] + [b[1]], A[2] + [b[2]]]
    for i in range(3):
        max_row = i
        for k in range(i+1, 3):
            if abs(M[k][i]) > abs(M[max_row][i]):
                max_row = k
        M[i], M[max_row] = M[max_row], M[i]
        pivot = M[i][i]
        for j in range(i, 4):
            M[i][j] /= pivot
        for k in range(3):
            if k != i:
                factor = M[k][i]
                for j in range(i, 4):
                    M[k][j] -= factor * M[i][j]
    return [M[0][3], M[1][3], M[2][3]]

def is_outside_box(point, min_corner, max_corner):
    for i in range(3):
        if point[i] < min_corner[i] or point[i] > max_corner[i]:
            return True
    return False
```

### Edge Cases and Pitfalls
* **Zero Intersections:** If `numint` is 0 in a `LeafNode`, the code prints an error and sets the minimizer `mp` to the center of the cell `st + len/2` [S1, unofficial]. This prevents division by zero during the average point calculation [S1, unofficial].
* **Boundary Conditions:** The clamping logic ensures that the minimizer stays within the voxel bounds, which is critical for maintaining the topological correctness of the dual contouring mesh [S1, unofficial].

## Operating characteristics
* **Time Complexity:** Not given in the sources.
* **Memory Complexity:** Not given in the sources.
* **Measured Performance:** Not given in the sources.
* **Scaling:** Not given in the sources.
* **Strengths and Limitations:** Not given in the sources.
* **When to Use:** Not given in the sources.

## Background
The specific octree data structure and implementation details described here are from the GitHub repository `aewallin/dualcontouring` [S1, unofficial]. This implementation is an unofficial adaptation of the original algorithm, designed for educational and practical use in real-time rendering.

## Sources
[S1] aewallin/dualcontouring, https://github.com/aewallin/dualcontouring, fetched 2026-10-26 (unofficial)
