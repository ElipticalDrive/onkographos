# Analytic Tetrahedron Voxelization

Also called: 3D rasterization using tetrahedral mesh, Exact Conservative Voxelization of Tetrahedra, Graph-traversal voxelization, Remeshing-based voxelization, Tetrahedral Voxelization with Philipp B's Algorithm, analytic volume sampling of tetrahedra, voxelization of tetrahedra

## Overview
Analytic Tetrahedron Voxelization converts a continuous tetrahedral mesh into a discrete Cartesian grid representation by computing physically conservative moments (volume, linear, quadratic) for each voxel [S2, unofficial]. Unlike standard binary voxelization which only stores occupancy flags, this method ensures that the sum of voxelized moments exactly equals the analytic moments of the input tetrahedra, thereby avoiding aliasing noise and preserving physical quantities [S1, unofficial][S3, unofficial]. It distinguishes itself from sibling methods like [Monte Carlo 2D Voxelization](1734-monte-carlo-voxelization.md) by providing exact, deterministic results without stochastic error, and from [Analytic Moment Voxelization](3307-analytic-moment-voxelization.md) by specifically leveraging tetrahedral geometry and robust planar graph clipping for convex polyhedra intersection [S2, unofficial].

## How it works
The method operates in two primary modes: a point-in-tetrahedron rasterization approach for binary occupancy [S1, unofficial], and a conservative moment-based approach for physical quantities [S2, unofficial][S3, unofficial]. The conservative approach is the more general and robust implementation, detailed below.

### Inputs and Outputs
**Inputs:**
*   A tetrahedral mesh defined by vertices.
*   A destination Cartesian grid defined by dimensions $n = (n_x, n_y, n_z)$, cell sizes $d = (d_x, d_y, d_z)$, and polynomial order $p \in \{0, 1, 2\}$ [S3, unofficial].

**Outputs:**
*   A grid of moments `grid.moments` stored in row-major C order. The buffer size is at least $n_x n_y n_z$ for each moment component [S3, unofficial].
*   Orientation flags for grid points to assist in boundary detection [S3, unofficial].

### Data Structures
The implementation relies on specific structures to manage geometry and grid state [S3, unofficial]:

1.  **`r3d_plane`**: Defines a clip plane.
    *   `n`: Unit normal vector $(n_x, n_y, n_z)$.
    *   `d`: Signed distance from the origin to the plane.
2.  **`r3d_orientation`**: Stores the status of a point relative to clip planes.
    *   `fdist[4]`: Signed distances to the four tetrahedron faces.
    *   `fflags`: Bit flags (unsigned char) indicating if the point is inside/outside each clip plane.
3.  **`r3d_dest_grid`**: The output grid container.
    *   `n`: Grid dimensions $(n_x, n_y, n_z)$.
    *   `d`: Cell sizes $(d_x, d_y, d_z)$.
    *   `polyorder`: Integer $p$ (0=constant, 1=linear, 2=quadratic).
    *   `moments[10]`: Array of pointers to gridded moment buffers. For $p=2$, indices 0-9 correspond to constant, linear (x,y,z), and quadratic (xx, yy, zz, xy, xz, yz) moments.
    *   `orient`: Buffer for gridpoint orientation checks, size $(n_x+1)(n_y+1)(n_z+1)$.
    *   `bufsz`: Total buffer size.

### Algorithm Steps

#### 1. Preconditioning and Orientation
Ensure the tetrahedron vertices are oriented such that the signed volume is positive. If the determinant of the spanning vectors yields a negative volume, swap vertices 2 and 3 [S1, unofficial][S3, unofficial].
$$ V = \frac{\det(\vec{AB}, \vec{AC}, \vec{AD})}{6} $$
If $V < 0$, swap $C$ and $D$ [S1, unofficial].

#### 2. Face Representation
Convert the vertex representation to a face representation to obtain the four clip planes defining the tetrahedron [S3, unofficial]. Each face $f$ is defined by a unit normal $\mathbf{n}_f$ and signed distance $d_f$.

#### 3. Voxel Search
Identify voxels that potentially intersect the tetrahedron. Two search strategies are used [S2, unofficial]:
*   **Brute-force**: Check every grid point against each tetrahedron face. Scales as $g^3$.
*   **Binary Space Partitioning (BSP)**: Use a spatial tree to prune empty regions. Scales as $g^2 \log g$. BSP only overtakes brute-force at grid resolutions above $128^3$ [S2, unofficial].

#### 4. Point-in-Tetrahedron Test (Philipp B's Algorithm)
For each candidate voxel center $P$, determine if it lies inside the tetrahedron $ABCD$. Define vectors $\mathbf{a} = \vec{PA}$, $\mathbf{b} = \vec{PB}$, $\mathbf{c} = \vec{PC}$, $\mathbf{d} = \vec{PD}$ [S1, unofficial].
Compute determinants:
$$ D_a = \det(\mathbf{b}, \mathbf{c}, \mathbf{d}) $$
$$ D_b = \det(\mathbf{a}, \mathbf{c}, \mathbf{d}) $$
$$ D_c = \det(\mathbf{a}, \mathbf{b}, \mathbf{d}) $$
$$ D_d = \det(\mathbf{a}, \mathbf{b}, \mathbf{c}) $$
Point $P$ is inside if:
$$ (D_a > 0 \land D_b < 0 \land D_c > 0 \land D_d < 0) \lor (D_a < 0 \land D_b > 0 \land D_c < 0 \land D_d > 0) $$
This algorithm is measured to be faster than other point-in-tetrahedron tests [S1, unofficial].

#### 5. Conservative Moment Calculation (Clipping and Reduction)
For voxels intersecting the boundary, compute the exact intersection volume and moments using a planar graph representation [S2, unofficial].

1.  **Clipping**: Intersect the tetrahedron with the voxel cube by successively clipping the tetrahedron against the six faces of the cube. This is done using a depth-first graph traversal on the planar graph representation of the polyhedron. This ensures topological validity and geometric robustness without auxiliary checks or high-precision arithmetic [S2, unofficial]. The clipping operation inserts new vertices in the correct order on the fly [S2, unofficial].
2.  **Integration**: Calculate the moment integrals (constant, linear, quadratic) over the resulting clipped polyhedron. The integration routine is coupled to the clipping algorithm, operating on the same planar graph structure. This allows storing polyhedra using only vertex locations and neighbors, without needing face normals [S2, unofficial].
3.  **Reduction**: Add the computed moments to the corresponding grid cells in `grid.moments` [S3, unofficial].

### Pseudocode
The following pseudocode implements the conservative voxelization of a single tetrahedron onto a grid, based on the `r3d` library logic [S3, unofficial] and the clipping scheme [S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize_tetrahedron(verts, grid):
    """
    Voxelize a single tetrahedron onto a Cartesian grid.
    verts: list of 4 points [A, B, C, D]
    grid: r3d_dest_grid structure
    """
    # 1. Precondition: Ensure positive volume
    # Calculate signed volume determinant
    vol_det = determinant(verts[1]-verts[0], verts[2]-verts[0], verts[3]-verts[0])
    if vol_det < 0:
        # Swap C and D to ensure positive orientation
        verts[2], verts[3] = verts[3], verts[2]

    # 2. Convert to face representation (4 clip planes)
    faces = tet_faces_from_verts(verts)
    
    # 3. Search for relevant voxels
    # Use BSP or brute-force to find voxels intersecting the tet bounding box
    candidate_voxels = search_voxels(verts, grid)

    for voxel in candidate_voxels:
        # 4. Determine voxel status relative to tetrahedron
        # Check corners of the voxel cube against the 4 tetrahedron faces
        corner_orientations = []
        for corner in voxel.corners:
            orient = r3d_orientation()
            for f_idx, face in enumerate(faces):
                dist = dot(corner, face.n) + face.d
                orient.fdist[f_idx] = dist
                if dist > 0:
                    orient.fflags |= (1 << f_idx)
            corner_orientations.append(orient)

        # 5. Classify voxel
        # andcmp: bits set if ALL corners are inside (all fflags == 0x0f)
        # orcmp: bits set if ANY corner is inside
        andcmp = 0x0F
        orcmp = 0x00
        for orient in corner_orientations:
            andcmp &= orient.fflags
            orcmp |= orient.fflags

        if andcmp == 0x0F:
            # Entirely inside: add analytic volume/moments directly
            add_full_voxel_moments(voxel, grid)
        elif orcmp != 0x0F:
            # Entirely outside: skip
            continue
        else:
            # Boundary voxel: clip and integrate
            # Initialize unit cube connectivity
            pixel = init_unit_cube(voxel)
            
            # Assign orientations to pixel vertices
            for v_idx, v in enumerate(pixel.verts):
                v.orient = corner_orientations[v_idx]

            # Clipping: Intersect tetrahedron with voxel cube
            # Uses depth-first graph traversal on planar graph
            clipped_poly = clip_polyhedron(faces, pixel, corner_orientations)
            
            # Integration: Compute moments of clipped polyhedron
            moments = integrate_moments(clipped_poly, grid.polyorder)
            
            # Reduction: Add to grid
            reduce_moments_to_grid(moments, voxel, grid)

def add_full_voxel_moments(voxel, grid):
    """Add analytic moments for a voxel fully inside the tetrahedron."""
    locvol = grid.d.x * grid.d.y * grid.d.z
    center_x = voxel.i * grid.d.x + grid.d.x * 0.5
    center_y = voxel.j * grid.d.y + grid.d.y * 0.5
    center_z = voxel.k * grid.d.z + grid.d.z * 0.5
    
    # Constant moment (Volume)
    grid.moments[0][voxel.index] += locvol
    
    if grid.polyorder >= 1:
        # Linear moments
        grid.moments[1][voxel.index] += locvol * center_x
        grid.moments[2][voxel.index] += locvol * center_y
        grid.moments[3][voxel.index] += locvol * center_z
        
    if grid.polyorder >= 2:
        # Quadratic moments
        # xx
        grid.moments[4][voxel.index] += locvol * (grid.d.x**2 / 12 + center_x**2)
        # yy
        grid.moments[5][voxel.index] += locvol * (grid.d.y**2 / 12 + center_y**2)
        # zz
        grid.moments[6][voxel.index] += locvol * (grid.d.z**2 / 12 + center_z**2)
        # xy
        grid.moments[7][voxel.index] += locvol * center_x * center_y
        # xz
        grid.moments[8][voxel.index] += locvol * center_x * center_z
        # yz
        grid.moments[9][voxel.index] += locvol * center_y * center_z

def integrate_moments(poly, polyorder):
    """
    Integrate moments over a clipped polyhedron represented by a planar graph.
    This function couples with the clipping routine.
    """
    # Implementation details depend on the specific integration scheme
    # (e.g., decomposition into pyramids from origin or divergence theorem).
    # The source notes state this is coupled to the clipping graph traversal.
    # For the purpose of implementation, this returns an array of moments.
    moments = [0.0] * 10
    # ... complex integration logic over the planar graph faces ...
    return moments
```

### Edge Cases and Pitfalls
*   **Geometric Degeneracy**: The method is immune to geometric degeneracies (e.g., vertices lying on clip planes) because it uses planar graph representations and depth-first traversal, avoiding auxiliary checks [S2, unofficial].
*   **Cancellation Errors**: Shifting voxels to a local coordinate frame prior to integration can cause cancellation errors, reducing accuracy to a few orders of magnitude below machine precision, although the reduction process itself is accurate to machine precision [S2, unofficial].
*   **Input Mesh Quality**: The rasterization variant (Philipp B's algorithm) assumes high-quality tetrahedral meshes with large minimum angles, no overlapping elements, and small variance in element quality [S1, unofficial].
*   **Search Performance**: Brute-force search is faster than BSP for coarse grids due to overhead; BSP only becomes superior at resolutions above $128^3$ [S2, unofficial].

## Subtopics

### [Exact General Remeshing Scheme for Conservative Voxelization](3774-exact-general-remeshing-scheme-for-conservative-voxelization.md)
This method voxelizes a single tetrahedron onto a Cartesian grid by computing physically conservative moments (volume, linear, quadratic) for a specified polynomial order. It ensures that the total mass in each grid cell exactly equals the integral of the input density field over the cell, avoiding aliasing noise. Unlike standard rasterization or Sutherland-Hodgman clipping, which can fail due to geometric degeneracy and roundoff errors, this scheme uses planar graph representations to maintain topological consistency during clipping, guaranteeing robustness even when vertices lie on clip planes.

## Operating characteristics
*   **Time Complexity**: Clipping and reduction operations scale roughly quadratically with the grid dimension $g$ ($O(g^2)$), corresponding to the surface area of the tetrahedra in units of squared grid spacing [S2, unofficial]. Brute-force search scales as $O(g^3)$, while Binary Space Partitioning scales as $O(g^2 \log g)$ [S2, unofficial].
*   **Memory Complexity**: The output grid requires storage for moments proportional to the number of voxels $N = n_x n_y n_z$. For polynomial order 2, 10 moment components are stored per voxel [S3, unofficial].
*   **Accuracy**: The method is "physically conservative," meaning the sum of voxelized moments equals the analytic moments of the tetrahedron up to floating-point precision [S3, unofficial]. Maximum fractional errors for random tetrahedra on a $128^3$ grid are approximately $10^{-12}$ for constant moments, $10^{-10}$ for linear moments, and $10^{-4}$ for quadratic moments in double precision [S2, unofficial].
*   **Performance**: The proposed clipping operation is 3-4 times faster than López and Hernández (2008), yielding an overall speed-up of a factor of 2 [S2, unofficial]. Philipp B's algorithm is faster than other point-in-tetrahedron tests [S1, unofficial].
*   **Strengths**: Robustness to geometric degeneracy, exact conservation of physical quantities, and high performance for convex polyhedra.
*   **Limitations**: Requires high-quality input tetrahedral meshes for the rasterization variant. Cancellation errors in local coordinate shifting can affect high-order moment accuracy [S2, unofficial].

## Background
Analytic Tetrahedron Voxelization originates from the need for exact, conservative discretization of continuous fields in scientific computing, particularly in cosmological N-body systems [S2, unofficial]. The clipping and integration framework is based on the ideas of Sugihara (1994), who described using planar graph representations for geometrically robust clipping [S2, unofficial]. The point-in-tetrahedron test known as "Philipp B's algorithm" was proposed by Philipp Beisel in June 2020 in response to a question by Denis Lolik, and was proven correct in the paper by [S1, unofficial]. The `r3d` library implementation provides a C-based solution for this conservative voxelization [S3, unofficial].

## Sources
[S1] Robust Voxelization and Visualization by Improved Tetrahedral Mesh Generation, https://doi.org/10.48550/arxiv.2106.01326, fetched 2026-10-26 (unofficial)
[S2] An exact general remeshing scheme applied to conservative voxelization, https://arxiv.org/pdf/1412.4941, fetched 2026-10-26 (unofficial)
[S3] devonmpowell/r3d, https://github.com/devonmpowell/r3d, fetched 2026-10-27 (unofficial)
