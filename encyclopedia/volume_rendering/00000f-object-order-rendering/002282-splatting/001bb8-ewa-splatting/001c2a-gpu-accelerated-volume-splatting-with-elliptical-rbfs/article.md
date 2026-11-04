# GPU-Accelerated Volume Splatting With Elliptical RBFs

Also called: Ellipsoidal Kernel Slicing with Re-orthogonalization, Elliptical RBF Slicing, Elliptical RBF Splatting, Modified Image-Aligned Splatting with Elliptical RBFs, Post-shaded kernel-centric rendering, RBF Volume Splatting, Screen-aligned Ellipsoid Splatting

## Overview
This method renders scattered volumetric data represented as 3D ellipsoidal Radial Basis Functions (RBFs) using a post-shaded, kernel-centric approach [S1, unofficial]. It extends high-quality image-aligned sheet-buffer splatting to irregular grids by slicing kernels into parallel, image-aligned density planes, rasterizing them as 2D footprints, and compositing them [S1, unofficial]. Unlike fragment-centric methods that evaluate Gaussian functions per pixel, this technique leverages hard-wired GPU polygonal rasterization, achieving two orders of magnitude higher frame rates [S1, unofficial]. It differs from its parent [EWA Splatting](1069-ewa-splatting.md) by supporting general ellipsoidal kernels and using a flat spatial decomposition with active cell lists rather than screen-aligned spherical impostors [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of scattered 3D points, each associated with an ellipsoidal RBF defined by center, orientation, and scale.
*   **Output:** A 2D image composited from front-to-back or back-to-front.

**Data Structures:**
1.  **Spatial Decomposition:** A low-resolution 3D regular cubic grid (not an octree). Each cell maintains a list of intersecting RBFs [S1, unofficial].
2.  **Textures:**
    *   **2D Gaussian Footprint Texture:** Represents the in-plane shape of the splat [S1, unofficial].
    *   **1D Gaussian Scaling Texture:** Contains the scaling function $Ae^{-kz^2}$, indexed by distance from kernel center to slice plane [S1, unofficial].
3.  **Active Cell List:** Cells traversed in front-to-back order, containing vertex attribute arrays for rendering [S1, unofficial].

**Parameters:**
*   $A$: Amplitude.
*   $k$: Spread parameter.
*   $\Delta z$: Sampling distance in screen space, typically 1.0 [S1, unofficial].

**Algorithm Steps:**

1.  **Preprocessing (CPU):**
    1.  Load volume data.
    2.  Slice and rasterize every point onto the low-resolution 3D grid. Each grid point maintains a list of all intersecting points [S1, unofficial].
    3.  Traverse the 3D cubic structure in front-to-back order to return a list of data points for each non-empty cell [S1, unofficial].
    4.  Flatten the structure onto a set of four 2D textures by copying the contents of every cell [S1, unofficial].
    5.  Process the resulting texture with the ellipsoid slicing fragment program to generate attribute vertex arrays for rendering [S1, unofficial].

2.  **Rendering Setup (Per Viewpoint Change):**
    1.  Place cells in an active list in front-to-back traversal order of the decomposition data structure [S1, unofficial].

3.  **Slicing and Rasterization (Per Active Cell):**
    1.  Pass the current slicing plane and current cell ID as uniform variables to the vertex program [S1, unofficial].
    2.  For each data point in the cell, compute the ID of the starting cell (where the first slicing polygon is encountered) by projecting $(startx, starty, startz)$ to the 3D cubic decomposition structure [S1, unofficial].
    3.  Assign the starting cell ID to the point in all cells it participates in [S1, unofficial].
    4.  Compute the cell ID for each cell sliced by the current sheet buffer.
    5.  **Culling:** Allow a point/vertex to render only if its starting cell ID matches the current cell ID AND it intersects the current slicing buffer. If the starting cell ID differs, the point has already been activated and is culled [S1, unofficial].
    6.  Call each active cell's associated vertex list using `glDrawArrays` [S1, unofficial].
    7.  Remove cells from the active list when `currentSliceZ` has passed even the largest point, determined by comparing with the maximum point diagonal for the cell [S1, unofficial].

4.  **Kernel Slicing (Per Ellipsoid):**
    1.  Decompose the 3D Gaussian kernel $G$ into a scaling component and a 2D footprint using separability [S1, unofficial]:
        $$G = Ae^{-k(x^2+y^2+z^2)} = Ae^{-kz^2} e^{-k(x^2+y^2)}$$
        Here, $Ae^{-kz^2}$ is the scaling function and $e^{-k(x^2+y^2)}$ is the splat component [S1, unofficial].
    2.  Compute the inverse transformation matrix $M'^{-1}$:
        $$M' = R_V \cdot R \cdot S$$
        $$M'^{-1} = S^{-1} \cdot R^{-1} \cdot R_V^{-1}$$
        where $R_V$ is view rotation, $R$ is object rotation, and $S$ is scaling [S1, unofficial].
    3.  Extract the first three columns of $M'^{-1}$ as vectors $u, v, n$:
        $$[u, v, n] = M'^{-1}$$
        These vectors are generally non-orthogonal [S1, unofficial].
    4.  **Re-orthogonalization:** Compute corrected normalized vectors $u', v', n'$:
        $$n' = \text{normalize}(u \times v)$$
        $$u' = \text{normalize}(n' \times n)$$
        $$v' = \text{normalize}(n' \times u)$$
        [S1, unofficial]
    5.  Compute the final normal vector $n''$ for all slicing planes in unit sphere space:
        $$n'' = \|n \cdot n'\| \cdot n'$$
        The length of $n''$ represents the distance between slicing planes [S1, unofficial].
    6.  Define the slicing polygon vertices in unit sphere space ($slice_{us}$):
        $$slice_{us} = \{-u' - v', -u' + v', u' + v', u' - v'\}$$
        The normal vector in unit sphere space is $n_{us} = n''$ [S1, unofficial].
    7.  Transform the slicing polygon to screen space:
        $$slice_{SS} = M \cdot slice_{US}$$
        Transform the normal vector to screen space:
        $$n_{ss} = M \cdot n_{us}$$
        [S1, unofficial]
    8.  **Incremental Slicing:** Perform the (screen space - unit sphere space) transformation once for the initial kernel mapping. Use simple vector additions to obtain remaining mappings. The unit sphere space slice distance vector is $n_{us} = (0, 0, n'')$. In unit texture space, the slice is defined as $slice_{us} = \{(0, 0, 0), (0, 1, 0), (1, 1, 0), (1, 0, 0)\}$ [S1, unofficial].

5.  **Rasterization and Shading:**
    1.  Index the 1D Gaussian scaling texture using the distance between the current slice plane and the kernel center. Retrieve the scaling factor [S1, unofficial].
    2.  Rasterize the 2D generic Gaussian footprint texture, scaled by the retrieved factor, onto the screen [S1, unofficial].
    3.  Perform color lookup and shading using interpolated densities within the slice $(x, y)$ and across the slice $(z)$, utilizing a transfer function [S1, unofficial].
    4.  Composite the rasterized slices using a front-to-back or back-to-front compositing buffer [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_volume(points, grid_resolution):
    """
    Preprocessing step performed on CPU.
    """
    grid = create_3d_grid(grid_resolution)
    for point in points:
        # Determine which grid cells the point's ellipsoid intersects
        intersecting_cells = get_intersecting_cells(point.ellipsoid, grid)
        for cell in intersecting_cells:
            cell.point_list.append(point)
    
    # Flatten grid to textures
    textures = flatten_grid_to_textures(grid)
    
    # Generate vertex attribute arrays
    vertex_arrays = process_textures_with_fragment_program(textures)
    return vertex_arrays

def render_frame(vertex_arrays, grid, view_matrix, model_matrix, scale_matrix):
    """
    Rendering loop.
    """
    # Compute transformation matrices
    R_V = view_matrix.rotation
    R = model_matrix.rotation
    S = scale_matrix
    
    # Compute inverse transformation for slicing
    M_prime_inv = invert(S) * invert(R) * invert(R_V)
    
    # Extract vectors
    u, v, n = extract_columns(M_prime_inv, 3)
    
    # Re-orthogonalize
    n_prime = normalize(cross(u, v))
    u_prime = normalize(cross(n_prime, n))
    v_prime = normalize(cross(n_prime, u))
    
    # Compute normal distance
    n_double_prime = dot(n, n_prime) * n_prime
    
    # Define unit sphere slice polygon
    slice_us = [
        vector(-u_prime.x - v_prime.x, -u_prime.y - v_prime.y, -u_prime.z - v_prime.z),
        vector(-u_prime.x + v_prime.x, -u_prime.y + v_prime.y, -u_prime.z + v_prime.z),
        vector(u_prime.x + v_prime.x, u_prime.y + v_prime.y, u_prime.z + v_prime.z),
        vector(u_prime.x - v_prime.x, u_prime.y - v_prime.y, u_prime.z - v_prime.z)
    ]
    
    # Transform to screen space
    M = view_matrix * model_matrix
    slice_ss = transform(M, slice_us)
    n_ss = transform(M, n_double_prime)
    
    # Rasterize
    rasterize_polygon(slice_ss, n_ss, scale_factor)

def slice_and_rasterize_cell(cell, current_slice_z, current_cell_id, vertex_arrays):
    """
    Process a single cell in the active list.
    """
    for point in cell.point_list:
        # Culling check
        if point.starting_cell_id != current_cell_id:
            continue
            
        if not intersects_slice(point.ellipsoid, current_slice_z):
            continue
            
        # Perform slicing and rasterization
        perform_kernel_slicing(point, current_slice_z)
        rasterize_kernel_slice(point)

def check_cell_completion(cell, current_slice_z):
    """
    Determine if a cell is done producing slices.
    """
    if current_slice_z > cell.max_point_diagonal:
        return True
    return False
```

## Operating characteristics
*   **Time Complexity:** Setup cost per ellipsoid involves two matrix-matrix multiplications, three cross-products, two normalizations, one dot product, and four matrix-vector multiplications [S1, unofficial]. Incremental slicing cost is four vector additions and one scalar addition [S1, unofficial].
*   **Memory Complexity:** Requires storage for the spatial decomposition grid and vertex attribute arrays. The flat spatial decomposition is less complex than an octree but may have higher memory overhead for sparse data [S1, unofficial].
*   **Measured Performance:** Tested on a Pentium 4 3GHz, 1 GB RAM, NVidia Quadro FX 3400 (256MB RAM, equivalent to GeForce 6800 GT) at 400x400 pixels resolution [S1, unofficial].
    *   Blunt Fin: 34k points (31k non-occluded), Diagonal 0.9-120, 1.6 FPS [S1, unofficial].
    *   Combustion: 39k points (35k non-occluded), Diagonal 8.3-27.4, 1.9 FPS [S1, unofficial].
    *   Chess Pc.: 84k points (63k non-occluded), Diagonal 1.9-2.5, 3.5 FPS [S1, unofficial].
    *   Monster: 91k points (60k non-occluded), Diagonal 2.5-3.0, 4.0 FPS [S1, unofficial].
    *   Toy Car: 173k points (117k non-occluded), Diagonal 2.0-4.3, 2.2 FPS [S1, unofficial].
    *   Toy Train: 199k points (139k non-occluded), Diagonal 2.0-4.0, 2.1 FPS [S1, unofficial].
*   **Scaling:** Achieves splat rendering rates two orders of magnitude higher than explicit kernel function evaluation approaches (e.g., Jang et al. reported 4 fps for <1000 kernels) [S1, unofficial].
*   **Strengths:** High performance due to hardware rasterization; post-shaded approach avoids color leakage and pre-shading artifacts; supports irregular grids [S1, unofficial].
*   **Limitations:** Elliptical datasets are significantly slower to rasterize than spherical ones due to larger diagonal ranges and overdraw from neighboring point overlaps [S1, unofficial]. Staircase artifacts may occur from point-sampling the slice function [S1, unofficial].
*   **When to use:** Suitable for subdivision volumes and computational science data encoded as irregularly distributed ellipsoidal RBFs [S1, unofficial].
*   **When not to use:** Not ideal for spherical RBFs where simpler methods might suffice, or when pre-integrated slab rendering is required to avoid staircase artifacts without additional processing [S1, unofficial].

## Background
The method was introduced by Neophytos Neophytou, Klaus Mueller, Kevin T. McDonnell, Wei Hong, Xin Guan, Hong Qin, and Arie Kaufman [S1, unofficial]. It extends previous work on GPU-accelerated volume splatting [NM05] and image-aligned splatting [MSH*99] to handle generalized ellipsoidal RBF kernels for scattered volume data [S1, unofficial]. It is used for rendering subdivision volumes and computational science data encoded as irregularly distributed ellipsoidal RBFs, such as those obtained by fitting Gaussian kernels to a mid-level of the subdivision hierarchy or fitting basis functions to a local Delauney triangulation [S1, unofficial].

## Sources
[S1] GPU-Accelerated Volume Splatting With Elliptical RBFs, https://doi.org/10.2312/vissym/eurovis06/013-020, fetched 2026-10-26 (unofficial)
