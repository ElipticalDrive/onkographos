# Analytic Tetrahedron Voxelization

Also called: 3D rasterization using tetrahedral mesh, Exact Conservative Voxelization of Tetrahedra, Graph-traversal voxelization, Remeshing-based voxelization, Tetrahedral Voxelization with Philipp B's Algorithm, analytic volume sampling of tetrahedra, voxelization of tetrahedra

## Overview
Analytic Tetrahedron Voxelization converts a parametrically defined continuous tetrahedral mesh into a digital voxel representation by computing physically conservative moments (volume, linear, quadratic) for each grid cell [S1, S2, S3, unofficial]. Unlike standard binary voxelization which only stores occupancy, this method ensures that the total mass in each grid cell exactly equals the integral of the input density field over the cell, avoiding aliasing noise [S3, unofficial]. It sets itself apart from [Monte Carlo 2D Voxelization](1734-monte-carlo-2d-voxelization.md) and [Differentiable Voxelization (MC/CF)](1742-differentiable-voxelization-mc-cf.md) by providing exact, deterministic geometric intersection results rather than statistical approximations, and from [Analytic Moment Voxelization](3307-analytic-moment-voxelization.md) by focusing specifically on the tetrahedral primitive and its robust clipping implementation [G].

## How it works
**Inputs and Outputs:**
*   **Input:** A tetrahedral mesh defined by vertices. The mesh must have high quality: large minimum angle, no overlapping elements, and small variance in element quality [S1, unofficial].
*   **Output:** A Cartesian grid containing moments. The grid stores `moments` (array of arrays for constant, linear, and quadratic orders) and `orient` (orientation flags) [S3, unofficial].

**Data Structures:**
The implementation relies on specific structures for grid storage and geometric representation [S3, unofficial]:

1.  `r3d_dest_grid`:
    *   `r3d_real* moments[10]`: Gridded moments in row-major C order. Size is at least $n_x \times n_y \times n_z$ [S3, unofficial].
    *   `r3d_int polyorder`: Polynomial order (0=constant, 1=linear, 2=quadratic).
    *   `r3d_orientation* orient`: Buffer for gridpoint orientation checks. Size is at least $(n_x+1) \times (n_y+1) \times (n_z+1)$ [S3, unofficial].
    *   `r3d_long bufsz`: Buffer size.
    *   `r3d_dvec3 n`: Grid dimensions $(n_x, n_y, n_z)$.

2.  `r3d_orientation`:
    *   `r3d_real fdist[4]`: Signed distances to the four clip planes.
    *   `unsigned char fflags`: Bit flags indicating inside/outside status for each of the 4 clip planes.

3.  `r3d_plane` (Face representation):
    *   `r3d_dvec3 n`: Unit normal vector.
    *   `r3d_real d`: Signed distance to the origin.

**Algorithm Steps:**

1.  **Preconditioning (Orientation):**
    Ensure the tetrahedron vertices are oriented such that the volume is positive [S3, unofficial]. Calculate the signed volume using the determinant of vectors spanning the tetrahedron [S1, unofficial]. If the orientation function returns negative, swap vertices 2 and 3 [S3, unofficial].
    $$ \text{Signed Volume} = \frac{\det(\vec{AB}, \vec{AC}, \vec{AD})}{6} $$
    Exchanging two rows in the determinant leads to opposite signed volume [S1, unofficial].

2.  **Face Conversion:**
    Convert the vertex representation to a face representation to obtain the four clip planes defining the tetrahedron [S3, unofficial].

3.  **Voxel Search:**
    Identify voxels that need processing. Two methods are described:
    *   **Brute-force:** Check every grid point against each tetrahedron face. Scales as $g^3$ [S2, unofficial].
    *   **Binary Space Partitioning (BSP):** Use a spatial tree. Scales as $g^2 \log g$ [S2, unofficial].
    *   *Note:* Brute-force performs better for coarser grids due to BSP overhead; BSP overtakes brute-force at grid resolution $128^3$ [S2, unofficial].

4.  **Voxel Classification (Inside/Outside/Boundary):**
    For each voxel, determine its relationship to the tetrahedron [S1, unofficial].
    *   **Fully Inside:** All 4 clip planes contain the voxel [S3, unofficial].
    *   **Fully Outside:** The voxel is outside at least one clip plane [S3, unofficial].
    *   **Boundary:** The voxel intersects the tetrahedron boundary.

    *Classification using Philipp B's Algorithm:*
    To test if a point $P$ lies inside tetrahedron $ABCD$, define vectors $\mathbf{a}=\vec{PA}, \mathbf{b}=\vec{PB}, \mathbf{c}=\vec{PC}, \mathbf{d}=\vec{PD}$ [S1, unofficial]. Calculate determinants:
    $$ D_a = \det(\mathbf{b}, \mathbf{c}, \mathbf{d}) $$
    $$ D_b = \det(\mathbf{a}, \mathbf{c}, \mathbf{d}) $$
    $$ D_c = \det(\mathbf{a}, \mathbf{b}, \mathbf{d}) $$
    $$ D_d = \det(\mathbf{a}, \mathbf{b}, \mathbf{c}) $$
    Point $P$ is inside if:
    $$ (D_a > 0 \land D_b < 0 \land D_c > 0 \land D_d < 0) \lor (D_a < 0 \land D_b > 0 \land D_c < 0 \land D_d > 0) $$
    [S1, unofficial]

5.  **Moment Calculation (Reduction):**
    *   **Fully Inside Voxels:** Compute analytic moments directly [S3, unofficial]. For a voxel at grid index $(i,j,k)$ with cell size $d$, the constant moment is volume $V=d^3$ [G]. Linear moments are $V \cdot (i+0.5)d$ [G]. Quadratic moments involve terms like $V \cdot \frac{1}{3} d^2 (1+3i+3i^2)$ [G].
    *   **Boundary Voxels:** Perform clipping and integration [S2, unofficial].

6.  **Clipping and Integration (Boundary Voxels):**
    For boundary voxels, intersect the voxel (a convex polyhedron, typically a cube) with the tetrahedron [S2, unofficial].
    *   **Clipping:** Clip the voxel against the four faces of the tetrahedron. This is done by successively clipping one polyhedron against the faces of the other [S2, unofficial].
    *   **Graph Traversal:** Store polyhedra using only vertex locations and neighbors [S2, unofficial]. Use a depth-first graph traversal on the planar graph representation to ensure topological validity and geometric robustness [S2, unofficial]. This method inserts new vertices in the correct order on the fly, requiring no post-clipping reordering or face normals [S2, unofficial].
    *   **Integration:** Calculate moment integrals (constant, linear, quadratic) over the resulting clipped intersection domain [S2, unofficial]. The results are accumulated into `grid.moments` [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def analytic_tetrahedron_voxelization(tet_vertices, grid_dims, cell_size, polyorder):
    """
    Voxelize a single tetrahedron onto a Cartesian grid.
    tet_vertices: list of 4 points [A, B, C, D]
    grid_dims: (nx, ny, nz)
    cell_size: d
    polyorder: 0, 1, or 2
    """
    # 1. Preconditioning: Ensure positive volume
    A, B, C, D = tet_vertices
    vec_AB = B - A
    vec_AC = C - A
    vec_AD = D - A
    signed_vol = determinant(vec_AB, vec_AC, vec_AD) / 6.0
    
    if signed_vol < 0:
        # Swap vertices 2 and 3 (C and D) to fix orientation [S3, unofficial]
        tet_vertices = [A, B, D, C]
        A, B, C, D = tet_vertices
        vec_AB = B - A
        vec_AC = C - A
        vec_AD = D - A
        signed_vol = determinant(vec_AB, vec_AC, vec_AD) / 6.0

    # 2. Convert to Face Representation (Clip Planes)
    # Each face is defined by a unit normal n and distance d to origin [S3, unofficial]
    faces = []
    def get_plane(p1, p2, p3):
        n = normalize(cross(p2 - p1, p3 - p1))
        d = -dot(n, p1)
        return {'n': n, 'd': d}
    
    faces.append(get_plane(B, C, D))
    faces.append(get_plane(A, C, D))
    faces.append(get_plane(A, B, D))
    faces.append(get_plane(A, B, C))

    # Initialize grid moments
    nx, ny, nz = grid_dims
    d = cell_size
    # moments array structure: moments[order][index]
    # For simplicity, assume a flat array or list of arrays
    moments = [0.0] * (nx * ny * nz * 10) # Max 10 moments (1 const + 3 lin + 6 quad)
    
    # 3. Search and Process Voxels
    # Using brute-force for clarity, though BSP is faster for large grids [S2, unofficial]
    for i in range(nx):
        for j in range(ny):
            for k in range(nz):
                # Define voxel corners (8 corners)
                corners = []
                for dx in [0, 1]:
                    for dy in [0, 1]:
                        for dz in [0, 1]:
                            x = i * d + dx * d
                            y = j * d + dy * d
                            z = k * d + dz * d
                            corners.append([x, y, z])
                
                # 4. Classify Voxel
                # Check if fully inside, fully outside, or boundary
                # Using corner orientations (fflags) as per r3d implementation [S3, unofficial]
                
                andcmp = 0x0F
                orcmp = 0x00
                
                for corner in corners:
                    fflags = 0x00
                    for f_idx, face in enumerate(faces):
                        # Signed distance from corner to plane
                        dist = dot(corner, face['n']) + face['d']
                        if dist > 0.0:
                            fflags |= (1 << f_idx)
                    
                    andcmp &= fflags
                    orcmp |= fflags
                
                # andcmp == 0x0F means all corners are inside (all bits 1)
                # orcmp != 0x0F means not all corners are inside.
                # If andcmp == 0x00, then ALL corners are outside -> Fully Outside.
                
                if andcmp == 0x00:
                    continue # Fully Outside [S3, unofficial]
                
                if andcmp == 0x0F:
                    # Fully Inside: Compute analytic moments directly [S3, unofficial]
                    idx = (i * ny * nz + j * nz + k)
                    vol = d * d * d
                    
                    # Constant moment
                    moments[idx * 10 + 0] += vol
                    
                    if polyorder >= 1:
                        # Linear moments [G]
                        cx = (i + 0.5) * d
                        cy = (j + 0.5) * d
                        cz = (k + 0.5) * d
                        moments[idx * 10 + 1] += vol * cx
                        moments[idx * 10 + 2] += vol * cy
                        moments[idx * 10 + 3] += vol * cz
                        
                    if polyorder >= 2:
                        # Quadratic moments [G]
                        moments[idx * 10 + 4] += vol * (1.0/3.0) * d * d * (1 + 3*i + 3*i*i)
                        moments[idx * 10 + 5] += vol * (1.0/3.0) * d * d * (1 + 3*j + 3*j*j)
                        moments[idx * 10 + 6] += vol * (1.0/3.0) * d * d * (1 + 3*k + 3*k*k)
                        moments[idx * 10 + 7] += vol * 0.25 * d * d * (1 + 2*i) * (1 + 2*j)
                        moments[idx * 10 + 8] += vol * 0.25 * d * d * (1 + 2*i) * (1 + 2*k)
                        moments[idx * 10 + 9] += vol * 0.25 * d * d * (1 + 2*j) * (1 + 2*k)
                    continue
                
                # Boundary: Clip and Integrate [S2, unofficial]
                # Full graph-traversal clipping routine is not provided in the sources [S2, unofficial].
                # In practice, this step successively clips the voxel cube against the 4 tetrahedron faces
                # using a depth-first traversal on a planar graph representation, then integrates moments.
                pass

    return moments

def determinant(v1, v2, v3):
    return v1[0]*(v2[1]*v3[2] - v2[2]*v3[1]) - v1[1]*(v2[0]*v3[2] - v2[2]*v3[0]) + v1[2]*(v2[0]*v3[1] - v2[1]*v3[0])

def cross(v1, v2):
    return [
        v1[1]*v2[2] - v1[2]*v2[1],
        v1[2]*v2[0] - v1[0]*v2[2],
        v1[0]*v2[1] - v1[1]*v2[0]
    ]

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def normalize(v):
    mag = (v[0]**2 + v[1]**2 + v[2]**2)**0.5
    return [v[0]/mag, v[1]/mag, v[2]/mag]
```

## Subtopics

### [Exact General Remeshing Scheme for Conservative Voxelization](3774-exact-general-remeshing-scheme-for-conservative-voxelization.md)
This method voxelizes a single tetrahedron onto a Cartesian grid by computing physically conservative moments (volume, linear, quadratic) for a specified polynomial order. It ensures that the total mass in each grid cell exactly equals the integral of the input density field over the cell, avoiding aliasing noise. Unlike standard rasterization or Sutherland-Hodgman clipping, which can fail due to geometric degeneracy and roundoff errors, this scheme uses planar graph representations to maintain topological consistency during clipping, guaranteeing robustness even when vertices lie on clip planes.

### [Platonic Solid Voxelization](16069-platonic-solid-voxelization.md)
Platonic Solid Voxelization generates a 3D binary voxel array representing the interior of regular platonic solids (tetrahedron, cube, octahedron, dodecahedron, and icosahedron). It outputs a 3D array where the voxel value is 1 if inside the polyhedron and 0 otherwise. Unlike [Analytic Tetrahedron Voxelization](3724-analytic-tetrahedron-voxelization.md), which computes conservative moments for arbitrary tetrahedra, this method outputs a simple binary mask for specific regular polyhedra. It differs from [Exact General Remeshing Scheme for Conservative Voxelization](3774-exact-general-remeshing-scheme-for-conservative-voxelization.md) by using a direct half-space intersection test rather than remeshing.

## Operating characteristics
**Time and Memory Complexity:**
*   Clipping and reduction operations scale roughly quadratically with the grid dimension $g$ (linearly with surface area) [S2, unofficial].
*   Brute-force search scales as $g^3$ [S2, unofficial].
*   Binary space partitioning (BSP) search scales as $g^2 \log g$ [S2, unofficial].
*   Memory usage is dominated by the grid storage for moments and orientation flags [S3, unofficial].

**Measured Performance:**
*   On a $128^3$ grid with $10^5$ tetrahedra, the method produces $1.3 \times 10^4$ interior and $1.5 \times 10^4$ boundary voxels per tetrahedron on average [S2, unofficial].
*   The proposed clipping operation is 3-4 times faster than López and Hernández (2008), with an overall speed-up of 2x [S2, unofficial].
*   Philipp B's algorithm is faster than other point-in-tetrahedron algorithms [S1, unofficial].
*   BSP overtakes brute-force at grid resolution $128^3$ [S2, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Physically conservative (sum of voxel moments equals analytic moments up to floating-point precision) [S3, unofficial]. Immune to geometric degeneracies (e.g., vertices on clip planes) [S2, unofficial]. Does not require watertight or topologically closed meshes [S1, unofficial].
*   **Limitations:** Cancellation errors arise when shifting voxels to a local coordinate frame, causing accuracy to be a few orders of magnitude away from machine precision despite the reduction process being accurate to machine precision [S2, unofficial]. Requires high-quality input tetrahedral meshes (large minimum angle, no overlapping) [S1, unofficial].

**When to Use:**
*   Use when physical conservation of mass/moments is critical (e.g., cosmological N-body systems) [S2, unofficial].
*   Use when exact geometric robustness is required for degenerate cases [S2, unofficial].
*   Do not use if binary occupancy is sufficient and performance is paramount for very coarse grids (brute-force is simple) [S2, unofficial].

## Background
Analytic Tetrahedron Voxelization originates from the need for exact, mass-conservative voxelization in cosmological N-body systems [S2, unofficial]. The specific clipping and integration framework was developed to address geometric degeneracies in polyhedron intersection, building on ideas from Sugihara (1994) regarding planar graph representations [S2, unofficial]. The point-in-tetrahedron test known as "Philipp B's algorithm" was introduced by Philipp Beisel in June 2020 in response to a question by Denis Lolik, and was proven correct in the paper by [S1, unofficial]. The C implementation `r3d` is available from Los Alamos National Laboratory [S3, unofficial].

## Sources
[S1] Robust Voxelization and Visualization by Improved Tetrahedral Mesh Generation, https://doi.org/10.48550/arxiv.2106.01326, fetched 2026-10-26 (unofficial)
[S2] An exact general remeshing scheme applied to conservative voxelization, https://arxiv.org/pdf/1412.4941, fetched 2026-10-26 (unofficial)
[S3] devonmpowell/r3d, https://github.com/devonmpowell/r3d, fetched 2026-10-27 (unofficial)
