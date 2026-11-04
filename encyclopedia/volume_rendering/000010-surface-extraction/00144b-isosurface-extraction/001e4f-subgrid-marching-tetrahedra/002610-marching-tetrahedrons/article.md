# Marching Tetrahedrons

Also called: IsosurfaceTet, Marching Tetrahedra (Case-based), Tetrahedral Marching

## Overview
Marching Tetrahedrons is an isosurface extraction algorithm that decomposes each voxel of a 3D grid into tetrahedra to determine the intersection of a scalar field with a target isovalue [S1, S3, unofficial]. Unlike its sibling [Marching Cubes](4328-naive-isosurface-extraction.md), which suffers from topological ambiguities in specific configurations, this method guarantees a manifold, intersection-free surface by resolving these ambiguities through tetrahedral decomposition [S1, S2, unofficial]. It produces a mesh that matches voxel data more accurately than Marching Cubes, at the cost of a higher vertex count [S2, unofficial].

## How it works
**Inputs:**
- A scalar field defined on a grid (structured or unstructured).
- An isovalue (threshold) $iso$ [S1, S3, unofficial].

**Outputs:**
- A list of triangles representing the isosurface [S1, unofficial].

**Data Structures:**
- **Tetrahedron:** A struct containing four vertices. Each vertex has 3D coordinates $(X, Y, Z)$ and a scalar field value $F$.
  - Fields: `X[4]`, `Y[4]`, `Z[4]` (float), `F[4]` (float) [S1, unofficial].
- **Lookup Table (`lup`):** A 16x6 integer array mapping case indices to edge pairs. `lup[icase][e]` stores the index of the $e$-th edge pair for case `icase`. Edge indices are defined as: 0=(0,1), 1=(1,2), 2=(0,2), 3=(1,3), 4=(2,3), 5=(0,3). If no segment exists, the value is -1 [S1, unofficial].
- **Segment Count Table (`numSegments`):** An array of size 16 mapping case indices to the number of segments (triangles or quads) to generate [S1, unofficial].

**Edge Definition:**
The tetrahedron has 6 edges connecting its 4 vertices (indices 0-3):
- Edge 0: Vertex 0 to Vertex 1
- Edge 1: Vertex 1 to Vertex 2
- Edge 2: Vertex 0 to Vertex 2
- Edge 3: Vertex 1 to Vertex 3
- Edge 4: Vertex 2 to Vertex 3
- Edge 5: Vertex 0 to Vertex 3

**Algorithm Steps:**

1. **Iterate Cells:** For each cell in the input grid, if it is a tetrahedron (4 vertices), extract its coordinates and scalar values into a `Tetrahedron` struct [S1, unofficial].

2. **Compute Case Index:** Determine the binary case index `icase` (0-15) based on the scalar values $F[i]$ relative to the isovalue $iso$.
   - If $F[i] < iso$, the $i$-th bit is 0.
   - If $F[i] \ge iso$, the $i$-th bit is 1.
   - $icase = \sum_{i=0}^{3} bit_i \cdot 2^i$ [S1, unofficial].
   - *Note:* Some implementations classify vertices as "inside" ($v \le iso$) or "outside" [S3, unofficial]. The bit logic must be consistent with the lookup table. The provided source [S1, unofficial] uses $F[i] < iso$ for bit 0.

3. **Lookup Configuration:**
   - Retrieve the number of segments: $n = numSegments[icase]$ [S1, unofficial].
   - If $n=0$, no surface intersects this tetrahedron. Continue to next cell.

4. **Compute Intersections:**
   - Retrieve the edge indices from `lup[icase]`.
   - For each edge index $e$ in the lookup table (until -1 is reached), compute the intersection point on the corresponding edge using linear interpolation [S1, S3, unofficial].
   - Let the edge connect vertex $v_0$ (scalar $F_0$, position $P_0$) and vertex $v_1$ (scalar $F_1$, position $P_1$).
   - Calculate the interpolation factor $t$:
     $$ t = \frac{iso - F_0}{F_1 - F_0} $$
   - Handle division by zero: If $|F_1 - F_0| < \epsilon$ (where $\epsilon = 10^{-6}$), set $t = 0.5$ or clamp based on context [S3, unofficial].
   - Clamp $t$ to $[0, 1]$ to prevent extrapolation outside the edge segment [S3, unofficial].
   - Calculate the intersection point $P_{int}$:
     $$ P_{int} = P_0 + t \cdot (P_1 - P_0) $$
   - Store the resulting points in temporary arrays (e.g., `x[]`, `y[]`, `z[]` for 1 triangle; `x1[]`, `y1[]`, `z1[]` for 2 triangles) [S1, unofficial].

5. **Generate Triangles:**
   - **Case $n=1$ (Triangle):** The lookup table specifies 3 edges. The 3 intersection points form a single triangle. Add it to the output list [S1, unofficial].
   - **Case $n=2$ (Quad):** The lookup table specifies 4 edges. The 4 intersection points form a quadrilateral. Triangulate the quad into two triangles. The connectivity depends on `icase` [S1, unofficial]:
     - If $icase \in \{5, 10\}$: Connect vertices 0-2-3 and 0-1-2 (or equivalent diagonal). Specifically, the source code adds triangles `(0,1,2)` and `(0,2,3)`? No, the source says: "for icase 5 or 10, connect vertices 0-2-3". Let's look at the code:
       ```python
       tl.AddTriangle(x1[0], y1[0], z1[0], x1[1], y1[1], z1[1], x1[2], y1[2], z1[2])
       if (icase == 5) or (icase == 10):
           tl.AddTriangle(x1[0], y1[0], z1[0], x1[2], y1[2], z1[2], x1[3], y1[3], z1[3])
       else: # icase 3, 12, 6, 9
           tl.AddTriangle(x1[1], y1[1], z1[1], x1[2], y1[2], z1[2], x1[3], y1[3], z1[3])
       ```
       This implies two different diagonals are used to maintain manifold connectivity across shared faces in the grid.
       - For $icase \in \{5, 10\}$: Triangles are $(0,1,2)$ and $(0,2,3)$.
       - For $icase \in \{3, 12, 6, 9\}$: Triangles are $(0,1,2)$ and $(1,2,3)$.
       - *Note:* The exact vertex ordering in the lookup table determines the indices 0-3. The source [S1, unofficial] code assumes the points are stored in the order they appear in `lup`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_tetrahedrons(tetrahedron, isovalue, lup, num_segments):
    """
    Extracts isosurface triangles from a single tetrahedron.
    
    Args:
        tetrahedron: Object with attributes X[4], Y[4], Z[4], F[4] (floats).
        isovalue: Scalar threshold.
        lup: 16x6 integer array mapping case index to edge pairs.
        num_segments: Array of size 16 mapping case index to segment count.
        
    Returns:
        List of triangles, where each triangle is a tuple of 3 (x, y, z) points.
    """
    EPS = 1e-6
    triangles = []
    
    # 1. Compute Case Index
    icase = 0
    for i in range(4):
        if tetrahedron.F[i] >= isovalue:
            icase |= (1 << i)
            
    # 2. Lookup Segment Count
    n_segments = num_segments[icase]
    
    if n_segments == 0:
        return triangles
        
    # 3. Compute Intersection Points
    # Edge mapping: 0:(0,1), 1:(1,2), 2:(0,2), 3:(1,3), 4:(2,3), 5:(0,3)
    edge_map = [
        (0, 1), (1, 2), (0, 2), (1, 3), (2, 3), (0, 3)
    ]
    
    points = []
    for e_idx in lup[icase]:
        if e_idx == -1:
            break
            
        v0_idx, v1_idx = edge_map[e_idx]
        v0 = (tetrahedron.X[v0_idx], tetrahedron.Y[v0_idx], tetrahedron.Z[v0_idx])
        v1 = (tetrahedron.X[v1_idx], tetrahedron.Y[v1_idx], tetrahedron.Z[v1_idx])
        f0 = tetrahedron.F[v0_idx]
        f1 = tetrahedron.F[v1_idx]
        
        # Linear Interpolation
        denom = f1 - f0
        if abs(denom) < EPS:
            t = 0.5
        else:
            t = (isovalue - f0) / denom
            t = max(0.0, min(1.0, t)) # Clamp to [0, 1]
            
        px = v0[0] + t * (v1[0] - v0[0])
        py = v0[1] + t * (v1[1] - v0[1])
        pz = v0[2] + t * (v1[2] - v0[2])
        points.append((px, py, pz))
        
    # 4. Generate Triangles
    if n_segments == 1:
        # 3 points form 1 triangle
        triangles.append((points[0], points[1], points[2]))
        
    elif n_segments == 2:
        # 4 points form 2 triangles (quad)
        p0, p1, p2, p3 = points[0], points[1], points[2], points[3]
        
        # Determine diagonal based on case index
        if icase in [5, 10]:
            # Diagonal 0-2: Triangles (0,1,2) and (0,2,3)
            triangles.append((p0, p1, p2))
            triangles.append((p0, p2, p3))
        else:
            # Diagonal 1-3: Triangles (0,1,2) and (1,2,3)
            # Note: Source code for else branch: AddTriangle(1,2,3) after adding (0,1,2)
            # Wait, source code:
            # tl.AddTriangle(x1[0]...x1[2]) -> (0,1,2)
            # else: tl.AddTriangle(x1[1]...x1[3]) -> (1,2,3)
            triangles.append((p0, p1, p2))
            triangles.append((p1, p2, p3))
            
    return triangles
```

## Subtopics

### [Adaptive Octree Marching Tetrahedra](3897-adaptive-octree-marching-tetrahedra.md)
Uses an adaptive octree to subdivide the volume, applying Marching Tetrahedrons only at leaf nodes where the isovalue is present. Reduces computation in empty regions compared to uniform grid traversal.

### [Naive Isosurface Extraction](4328-naive-isosurface-extraction.md)
Processes each tetrahedron independently without vertex sharing, resulting in non-manifold surfaces with cracks. Serves as a baseline for understanding the core decomposition logic.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of tetrahedra in the grid. Each tetrahedron is processed independently in $O(1)$ time.
- **Memory Complexity:** Linear with respect to the number of triangles generated. The output mesh size is typically larger than Marching Cubes for the same geometry due to the finer decomposition [S2, unofficial].
- **Performance:** Produces a mesh that matches voxel data better than Marching Cubes but creates significantly more vertices, making it slower and more memory-intensive [S2, unofficial].
- **Strengths:** Resolves topological ambiguities inherent in Marching Cubes (e.g., cases 6 and 7) by ensuring a consistent manifold surface through tetrahedral decomposition [S1, unofficial].
- **Limitations:** Higher vertex count leads to larger file sizes and increased rendering cost. Requires the input grid to be decomposable into tetrahedra (or an existing unstructured tetrahedral mesh) [S1, unofficial].
- **When to use:** When topological correctness is critical and the ambiguity of Marching Cubes is unacceptable, or when working with unstructured tetrahedral grids [S1, unofficial].
- **When not to use:** When performance and memory efficiency are paramount and the visual difference from Marching Cubes is negligible.

## Background
The method was developed as part of undergraduate science visualization coursework, utilizing C++, CMake, and VTK data files [S1, unofficial]. It is an alternative to Marching Cubes, designed to address the "tetrahedron ambiguity" where a single cube can be split in two topologically distinct ways. By decomposing the cube into tetrahedra, the algorithm ensures a unique, consistent surface for each configuration [S1, unofficial]. It is often implemented in graphics engines and visualization tools that require robust isosurface extraction from volumetric data [S2, S3, unofficial].

## Sources
[S1] adrian-sierra/Marching-Tetrahedrons, https://github.com/adrian-sierra/Marching-Tetrahedrons, fetched 2026-10-26 (unofficial)
[S2] Scrawk/Marching-Cubes, https://github.com/Scrawk/Marching-Cubes, fetched 2026-10-26 (unofficial)
[S3] Torbenhjl/Marching-Tetrahedra-on-CT-headscan, https://github.com/Torbenhjl/Marching-Tetrahedra-on-CT-headscan, fetched 2026-10-27 (unofficial)
