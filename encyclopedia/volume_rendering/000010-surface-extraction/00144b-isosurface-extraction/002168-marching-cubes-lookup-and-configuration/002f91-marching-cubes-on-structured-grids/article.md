# Marching Cubes on Structured Grids

Also called: MC extraction, Marching Cubes in IJK Coordinate Space, Marching Cubes mesh extraction from implicit volume

## Overview
Marching Cubes on Structured Grids extracts a 3D polygonal mesh from a discretized scalar field defined on a regular 3D grid (IJK space) by identifying the isosurface where the scalar value crosses a specified threshold [S1, unofficial][S2]. It builds on [Isosurface extraction](30-isosurface-extraction.md) by processing each voxel cell independently to determine local topology, distinguishing itself from unstructured methods by leveraging the predictable connectivity of the grid [S2]. Unlike [Advancing Front Framework for Implicit Surfaces](819-advancing-front-framework-for-implicit-surfaces.md) which grows the mesh from a seed, or [Dual Contouring and Surface Nets](6319-dual-contouring-and-surface-nets.md) which optimize vertex positions for sharp features, standard Marching Cubes relies on linear interpolation along cell edges [S2].

## How it works
**Inputs:**
- `grid`: A 3D array of scalar values (e.g., density, signed distance, neural network output) with dimensions $R_x \times R_y \times R_z$ [S1, unofficial].
- `c1`: Lower bounds of the volume in world coordinates $(x_{min}, y_{min}, z_{min})$ [S1, unofficial].
- `c2`: Upper bounds of the volume in world coordinates $(x_{max}, y_{max}, z_{max})$ [S1, unofficial].
- `isosurface`: The scalar threshold value $\sigma_{iso}$ defining the surface [S1, unofficial].
- `reso`: The resolution of the grid $(R_x, R_y, R_z)$ [S1, unofficial].

**Outputs:**
- `vertices`: An array of 3D points in world coordinates representing the mesh vertices [S1, unofficial].
- `triangles`: An array of integer indices referencing `vertices`, defining the mesh faces [S1, unofficial].

**Data Structures:**
- The scalar field is stored as a contiguous 3D array or flattened 1D array reshaped to 3D. In neural implicit contexts, this is often a tensor of shape $(R_x, R_y, R_z)$ [S1, unofficial].
- The standard Marching Cubes lookup table maps 8-bit cell configurations (defined by the sign of the 8 vertices relative to $\sigma_{iso}$) to a set of triangle indices [S2].

**Steps:**

1.  **Grid Generation**: Create a structured grid of query points in IJK coordinate space. If the grid is used to evaluate a neural network, generate points $P_{ijk}$ for $i \in [0, R_x-1], j \in [0, R_y-1], k \in [0, R_z-1]$ [S1, unofficial].
    $$ P_{ijk} = (i, j, k) $$
    [S1, unofficial]

2.  **Scalar Field Evaluation**: Evaluate the scalar field $\sigma$ at each grid point.
    - If using a neural network, pass $P_{ijk}$ through the network to obtain $\sigma_{ijk}$ [S1, unofficial].
    - If the network uses view directions, compute fake view directions (e.g., pointing to the origin) to avoid invalid geometry, as sigma becomes view-dependent otherwise [S1, unofficial].
    - Process in batches of size `eval_batch_size` to manage memory [S1, unofficial].
    $$ \sigma_{ijk} = \text{Network}(P_{ijk}) $$
    [S1, unofficial]

3.  **Reshape**: Reshape the flattened list of $\sigma_{ijk}$ values into the 3D grid structure of size $R_x \times R_y \times R_z$ [S1, unofficial].

4.  **Marching Cubes Extraction**: Apply the standard Marching Cubes algorithm to the scalar grid $\sigma$ with threshold $\sigma_{iso}$ [S1, unofficial].
    - For each cell $(i, j, k)$, determine the 8 vertex signs relative to $\sigma_{iso}$.
    - Use the lookup table to identify the triangle configuration.
    - Calculate intersection points on the cell edges using linear interpolation. For an edge between vertices $V_1$ (value $\sigma_1$) and $V_2$ (value $\sigma_2$), the intersection point $V_{int}$ in IJK space is:
      $$ V_{int} = V_1 + \frac{\sigma_{iso} - \sigma_1}{\sigma_2 - \sigma_1} (V_2 - V_1) $$
      [S2]

5.  **Coordinate Transformation**: Convert the extracted vertices from IJK grid space to World (XYZ) space.
    - Scale the IJK coordinates by the physical dimensions of the grid:
      $$ V_{world} = V_{ijk} \times \frac{c_2 - c_1}{\text{reso}} $$
      [S1, unofficial]
    - Translate by the lower bound:
      $$ V_{final} = V_{world} + c_1 $$
      [S1, unofficial]
    - Note: This linear interpolation from IJK to XYZ is an approximation; strictly speaking, it assumes a uniform Cartesian grid [S2]. For non-uniform structured grids, this step is not technically correct but is negligible for most applications [S2].

6.  **Return**: Output the final `vertices` and `triangles` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_cubes_on_structured_grid(network, c1, c2, reso, isosurface, eval_batch_size=100000):
    """
    Extracts a mesh from a scalar field defined on a structured grid.
    
    Args:
        network: A callable that returns scalar values for input points.
        c1: List[int] Lower bounds (x, y, z).
        c2: List[int] Upper bounds (x, y, z).
        reso: List[int] Resolution (Rx, Ry, Rz).
        isosurface: Float Threshold value.
        eval_batch_size: Int Number of points per batch.
        
    Returns:
        vertices: Array of shape (N, 3) in world coordinates.
        triangles: Array of shape (M, 3) with vertex indices.
    """
    # 1. Generate IJK grid
    # Create indices for the grid
    ix = np.arange(reso[0])
    iy = np.arange(reso[1])
    iz = np.arange(reso[2])
    # Create meshgrid and flatten to get all (i, j, k) coordinates
    ii, jj, kk = np.meshgrid(ix, iy, iz, indexing='ij')
    grid_points = np.stack([ii.ravel(), jj.ravel(), kk.ravel()], axis=-1).astype(np.float32)
    
    # 2. Evaluate Scalar Field
    all_sigmas = []
    
    # Split grid into batches
    grid_batches = np.array_split(grid_points, np.ceil(len(grid_points) / eval_batch_size))
    
    for batch in grid_batches:
        # If network requires viewdirs, compute fake viewdirs (pitfall mitigation)
        # Assuming network expects (points, viewdirs) or similar
        # Here we assume a simple scalar field evaluation for clarity
        sigmas = network(batch)
        all_sigmas.append(sigmas)
        
    # Concatenate and reshape
    sigmas = np.concatenate(all_sigmas, axis=0)
    sigmas = sigmas.reshape(reso)
    
    # 3. Run Marching Cubes
    # mcubes.marching_cubes expects the scalar volume and the threshold
    vertices_ijk, triangles = mcubes.marching_cubes(sigmas, isosurface)
    
    # 4. Transform to World Coordinates
    # Scale: (c2 - c1) / reso
    scale = (np.array(c2) - np.array(c1)) / np.array(reso)
    vertices_world = vertices_ijk * scale + np.array(c1)
    
    return vertices_world, triangles
```

**Edge Cases and Pitfalls:**
- **View Directions**: If the scalar field depends on view direction (e.g., NeRF), standard Marching Cubes is invalid because the surface is not well-defined without a camera position. Substituting fake view directions (e.g., pointing to the origin) may produce incorrect geometry [S1, unofficial].
- **IJK to XYZ Interpolation**: The transformation assumes a uniform grid. For non-uniform structured grids, this method is not technically correct, though the error is often negligible [S2].
- **Ambiguities**: Standard Marching Cubes may produce topological ambiguities in cells with specific configurations (e.g., saddle points). Advanced variants like [Matveyev's Algorithm](#matveyevs-algorithm) or [Extended CLCIR PLIC Reconstruction](954-extended-clcir-plic-reconstruction.md) address this, but standard MC uses a fixed lookup table [S3].

## Subtopics

### [Uniform Marching Cubes with Snapping](7468-uniform-marching-cubes-with-snapping.md)
Uniform Marching Cubes with Snapping is a uniform-grid polygonization algorithm that computes cell topology masks from coarse grid vertex signs and optionally snaps iso-surface vertices to edges by minimizing distance to grid vertices. It builds on [Marching Cubes on Structured Grids](6051-marching-cubes-on-structured-grids.md) by introducing a snapping mechanism that reduces geometric error, distinguishing it from standard UMC and other siblings like [Bitwise Marching Cubes](7374-bitwise-marching-cubes.md) which prioritize bitwise operations over positional accuracy.

### [Smart Conversion](7489-smart-conversion.md)
Smart Conversion is an advanced, gradient-based voxel-to-mesh conversion algorithm that prioritizes geometric accuracy by correcting errors arising from noise, multiple materials, or inaccurate iso-values. Unlike standard isosurface extraction methods such as [Marching Cubes on Structured Grids](6051-marching-cubes-on-structured-grids.md) which rely on fixed threshold crossings, Smart Conversion iteratively "pulls" the mesh vertices to align with the true boundary of the scan by locating points of steepest gradient along local surface normals. This makes it particularly suitable for medical or high-precision industrial contexts where the initial voxel data may contain flaws or where the iso-value does not perfectly represent the physical boundary.

### [Flying Edges](7520-flying-edges.md)
Flying Edges is a parallel, threaded algorithm for volumetric isocontouring of continuous scalar fields that addresses the performance limitations of Marching Cubes by processing volume edges rather than voxel cells. It eliminates point merging bottlenecks by using an edge-based iterator to generate output points and triangle cells, avoiding the need for hashing duplicate points.

### [Undeformed Space Ray Start via Deformed Hull Intersection](10141-undeformed-space-ray-start-via-deformed-hull-intersection.md)
This method finds the ray start point $\hat{x}_0 = D^{-1}(p)$ in undeformed space without computing the inverse deformation map $D^{-1}$ directly. It solves the problem where $D$ may not be invertible or well-defined at the camera position by intersecting the deformed view ray with a deformed explicit hull mesh that contains the surface.

### [Dividing Cubes](10391-dividing-cubes.md)
Dividing Cubes is an isosurface extraction method that replaces the triangular mesh primitives of Marching Cubes with 3D points. It solves the problem of excessive triangle counts and the subsequent scan-conversion overhead in Marching Cubes by adaptively subdividing cells intersected by the surface until each subcell projects onto a single pixel. Unlike its siblings, it generates a point cloud rather than a fixed mesh, allowing for gradient-shaded rendering without explicit triangle rasterization.

### [Analytic Marching](12525-analytic-marching.md)
Analytic Marching (AM) is a meshing algorithm that extracts the exact zero-level isosurface of an implicit function represented by a ReLU-based Multi-Layer Perceptron (MLP) as a polygon mesh, without relying on discrete 3D grid sampling. Unlike [Marching Cubes on Structured Grids](6051-marching-cubes-on-structured-grids.md), whose accuracy depends on discretization resolution, AM exploits the piecewise linear nature of ReLU networks to identify linear regions intersected by the surface and solve for vertices analytically, making its precision independent of grid resolution.

### [GPU-Accelerated Metaball Isosurface Generation](12526-gpu-accelerated-metaball-isosurface-generation.md)
This method generates an isosurface from a combined field of metaballs using a two-stage GPU pipeline, utilizing vertex shaders for field evaluation and geometry shaders for mesh generation. It is distinguished from standard CPU-based or single-pass GPU implementations by its explicit separation of field computation and mesh topology generation, coupled with Transform-Feedback caching to avoid recomputing geometry when the camera moves but the metaball configuration remains static. It builds on [Marching Cubes on Structured Grids](6051-marching-cubes-on-structured-grids.md) by offloading the structured grid vertex evaluation and cube processing to the GPU.

### Matveyev's Algorithm
An algorithm based on a triangulation table that correctly represents trilinear interpolant behavior using intersections on cube diagonals [S3]. Proposed by Matveyev in 1999, it resolves ambiguities in the standard Marching Cubes lookup table by considering the internal topology of the trilinear interpolant within the cube [S3].

### Geomorph LOD Seam Handling
A technique to achieve watertight seams between fine and coarse LOD levels in Marching Cubes by geometrically deforming the fine surface at the boundary to match the coarse contour [S4, unofficial]. It identifies boundary vertices between fine and coarse LOD levels and applies a geometric morph (bending) to fine surface vertices at the boundary, deforming them toward the coarse contour to eliminate gaps [S4, unofficial].

### Cell Subdivision Isosurface Extraction
A method that retains the identity of cells (vertex/node centered data) and eliminates non-linear hanging nodes by subdividing boundary cells into pyramids, allowing standard isosurface techniques to be applied [S5, unofficial]. It subdivides lower-resolution cell faces to match higher-resolution faces, obtains values at newly created hanging nodes by linear interpolation, and subdivides 3D cells into pyramids with a common apex point [S5, unofficial]. Unlike the dual grid approach, this method retains the original cell identity [S5, unofficial].

### TexF with Implicit Geometry (SDF)
An extension where TexF shares a canonical grid with an implicit Signed Distance Field (SDF). The surface is extracted via Marching Cubes, and TexF attributes are transferred to vertices for standard vertex-color rendering, avoiding per-pixel TexF queries [S6, unofficial]. The SDF and TexF share the same grid; after extracting a mesh with Marching Cubes, TexF attributes are transferred to its vertices using k-nearest-neighbor interpolation [S6, unofficial].

### 3-Edge Cube Sampling with Corner Extension
Instead of calculating intersection points for all 12 edges of a cube, compute intersections for only the 3 edges in the bottom-left corner of each cube [S7, unofficial]. Add an extra row in the top-right corner of the processing zone to ensure all necessary grid values are accessible for the reduced edge set [S7, unofficial]. The indexing logic requires a special upgrade/mapping to correctly access grid values, making it complicated to implement correctly [S7, unofficial].

### Octree-Based Isolevel Culling
Build an octree structure over the volume to assess the ISO level distribution, allowing the algorithm to skip processing areas far from the target isolevel [S7, unofficial]. Propagation-based culling methods are noted as unreliable and inefficient compared to direct octree assessment [S7, unofficial].

### Coarse Sampling Dead-Space Elimination
Perform an initial pass sampling the volume at a coarser spatial rate to identify and eliminate regions where field strength is well below the isolevel [S7, unofficial]. Coarse sampling provides virtually no performance gain for non-trivial fields, as safe sampling borders still capture most of the grid [S7, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N)$, where $N$ is the number of voxels in the grid, as each cell is processed independently [S2].
- **Memory Complexity**: $O(N)$ to store the scalar field and the output mesh. The algorithm can be implemented as a streaming operation where only two slices of the data are necessary in memory at any given time, allowing it to work on extremely large datasets [S2].
- **Performance**: Performance is heavily dependent on the cost of evaluating the scalar field at grid points. If the field evaluation is expensive (e.g., neural network inference), batching and GPU acceleration are critical [S1, unofficial].
- **Strengths**: Simple to implement, robust for smooth fields, and easily parallelizable due to cell independence [S2].
- **Limitations**: Produces triangular meshes which may not capture sharp features well without additional processing (e.g., [Dual Contouring and Surface Nets](6319-dual-contouring-and-surface-nets.md)). Sensitive to grid resolution; higher resolution yields better accuracy but increases memory and computation [S1, unofficial].
- **When to Use**: When the input data is naturally structured (e.g., CT scans, uniform grids) and the surface is relatively smooth [S2].
- **When Not to Use**: When the surface has sharp features requiring vertex optimization, or when the grid is highly non-uniform and the IJK-to-XYZ interpolation error is significant [S2].

## Background
The Marching Cubes algorithm was originally developed and patented by GE Medical Systems for use in their CT and MRI equipment [S2]. The patent has since expired, placing the algorithm in the public domain [S2]. It is one of the most common visualization methods in use today and is used by several other algorithms such as Metaballs and Part Extraction [S2]. The algorithm processes each cell individually, making it easy to convert into a "streaming" operation [S2].

## Sources
[S1] sxyu/pixel-nerf, https://github.com/sxyu/pixel-nerf, fetched 2026-10-26 (unofficial)
[S2] Marching Cubes Marching Cubes, https://daac.hpc.mil/gettingStarted/Marching_Cubes.html, fetched 2026-10-29
[S3] A Evolucao do Algoritmo Marching Cubes, https://doi.org/10.5540/03.2017.005.01.0107, fetched 2026-10-26
[S4] draquel/VoxelWorlds, https://github.com/draquel/VoxelWorlds, fetched 2026-10-27 (unofficial)
[S5] Visualization Tools for Adaptive Mesh Refinement Data, https://www.vacet.org/publications/AMRVisJune2007.pdf, fetched 2026-10-27 (unofficial)
[S6] Beyond UV Mapping: Mesh Texture Compression via Surface-Aligned Texture Fields, https://doi.org/10.48550/arxiv.2609.23606, fetched 2026-10-26 (unofficial)
[S7] How to speed up marching cubes?, https://stackoverflow.com/questions/868831/how-to-speed-up-marching-cubes, fetched 2026-10-26 (unofficial)
