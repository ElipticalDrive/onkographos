# Signed Euclidean Distance Field (SDF) Representation

Also called: Signed Distance Field (SDF), Signed Distance Function (SDF)

## Overview
A Signed Euclidean Distance Field (SDF) is a continuous implicit representation that maps any 3D point to a scalar value representing the shortest Euclidean distance to the nearest surface, with a sign indicating whether the point lies inside or outside the watertight boundary [S1, S2]. It differs from occupancy fields, which only indicate binary presence, by providing metric distance information that enables robust sphere tracing and smooth surface reconstruction [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D query point $\mathbf{p} \in \mathbb{R}^3$ and, in neural implementations, an observation context $\mathbf{x} \in \mathcal{X}$ (e.g., image data or identity parameters) [S1].
*   **Output:** A scalar value $s \in \mathbb{R}$ representing the signed distance [S1].

**Data Structures:**
*   **Grid-based:** A 3D voxel grid where each voxel stores a single floating-point value. The value is $-d$ for voxels outside the object boundary and $+d$ for voxels inside, where $d$ is the Euclidean distance to the nearest boundary [S2].
*   **Neural Implicit:** A parameterized function $f_\theta$ (e.g., a Multi-Layer Perceptron) that approximates the mapping [S1].

**Formulas:**
The fundamental mapping is defined as:
$$ s : \mathbb{R}^3 \rightarrow \mathbb{R} \quad [S1] $$

In the context of neural implicit representations conditioned on observations $\mathbf{x}$, the function is expressed as:
$$ f_\theta : \mathbb{R}^3 \times \mathcal{X} \rightarrow \mathbb{R} \quad [S1] $$

**Steps to Compute/Store:**
1.  **Define Boundary:** Identify the watertight surface boundary of the object [S1, S2].
2.  **Compute Distance:** For every point (voxel center or query location), calculate the Euclidean distance $d$ to the nearest point on the boundary [S2].
3.  **Assign Sign:**
    *   If the point is **outside** the boundary, assign value $-d$ [S2].
    *   If the point is **inside** the boundary, assign value $+d$ [S2].
    *   *Note on Neural Variants:* Some neural implementations (like O-Net) map to $[0,1]$ where the surface is the 0.5-level set, but the standard SDF definition maps to $\mathbb{R}$ with the zero-level set at 0 [S1].

**Parameters:**
*   **Resolution:** For grid-based SDFs, the voxel grid resolution determines the precision of the distance approximation [S2].
*   **Neural Parameters:** $\theta$ represents the weights and biases of the neural network approximating the field [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** In grid-based representations, the distance is only exact at voxel centers or interpolated values; aliasing can occur at high frequencies [S2].
*   **Non-Watertight Surfaces:** The definition assumes a watertight surface to define a clear inside/outside region [S1].

**Pseudocode (Grid-based Construction):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_sdf_grid(mesh, grid_resolution, bounds):
    """
    Computes a Signed Euclidean Distance Field on a 3D grid.
    
    Args:
        mesh: A watertight surface mesh defining the boundary.
        grid_resolution: Tuple (nx, ny, nz) defining the voxel count.
        bounds: Tuple (min_x, min_y, min_z, max_x, max_y, max_z) defining the grid extent.
        
    Returns:
        A 3D numpy array of shape (nx, ny, nz) containing signed distances.
    """
    nx, ny, nz = grid_resolution
    min_x, min_y, min_z, max_x, max_y, max_z = bounds
    
    # Initialize grid with zeros
    sdf_grid = np.zeros((nx, ny, nz))
    
    # Calculate voxel size
    dx = (max_x - min_x) / nx
    dy = (max_y - min_y) / ny
    dz = (max_z - min_z) / nz
    
    # Iterate over all voxels
    for i in range(nx):
        for j in range(ny):
            for k in range(nz):
                # Calculate center of the voxel in world space
                x = min_x + (i + 0.5) * dx
                y = min_y + (j + 0.5) * dy
                z = min_z + (k + 0.5) * dz
                point = np.array([x, y, z])
                
                # Compute Euclidean distance to the mesh boundary
                # This requires a nearest-point query on the mesh triangles
                distance_to_boundary = compute_euclidean_distance_to_mesh(point, mesh)
                
                # Determine if point is inside or outside
                # Using ray casting or winding number for point-in-mesh test
                is_inside = is_point_inside_mesh(point, mesh)
                
                if is_inside:
                    sdf_grid[i, j, k] = distance_to_boundary
                else:
                    sdf_grid[i, j, k] = -distance_to_boundary
                    
    return sdf_grid

def compute_euclidean_distance_to_mesh(point, mesh):
    """
    Computes the shortest Euclidean distance from a point to a mesh.
    [G] Implementation detail: Iterate over all triangles, compute distance 
    to triangle plane, clamp to triangle bounds, find minimum distance.
    """
    min_dist = infinity
    for triangle in mesh.triangles:
        dist = distance_point_to_triangle(point, triangle)
        if dist < min_dist:
            min_dist = dist
    return min_dist

def is_point_inside_mesh(point, mesh):
    """
    Determines if a point is inside a watertight mesh.
    [G] Implementation detail: Cast a ray from point to infinity, 
    count intersections with mesh triangles. Odd count implies inside.
    """
    ray_direction = np.array([1.0, 0.0, 0.0]) # Arbitrary direction
    intersections = 0
    for triangle in mesh.triangles:
        if ray_intersects_triangle(point, ray_direction, triangle):
            intersections += 1
    return intersections % 2 == 1
```

## Subtopics

### SDF Primitive Definitions
Defines analytical SDFs for basic shapes (spheres, boxes, cylinders) which serve as building blocks for complex scenes. Choose when procedural geometry is preferred over mesh-based SDFs. [SDF Primitive Definitions](1578-sdf-primitive-definitions.md)

### Distance Field Representations and Sampling
Discusses various ways to store and sample distance fields (e.g., sparse grids, hash tables) beyond dense voxel grids. Choose when memory efficiency is critical for large scenes. [Distance Field Representations and Sampling](8725-distance-field-representations-and-sampling.md)

### SDF Transformations
Details how to apply affine transformations (rotation, translation, scaling) to SDFs by transforming the query point. Choose when objects need to move or deform. [SDF Transformations](11027-sdf-transformations.md)

### Global Distance Query via Local-Global Blending
Addresses the challenge of computing accurate SDFs for large scenes by blending local high-resolution SDFs with a global low-resolution field. Choose for large-scale outdoor environments. [Global Distance Query via Local-Global Blending](14434-global-distance-query-via-local-global-blending.md)

### SDF Text Rendering
Specializes in generating SDFs from vector fonts for efficient text rendering. Choose when rendering text with smooth scaling and anti-aliasing. [SDF Text Rendering](18123-sdf-text-rendering.md)

### Implicit Surface Taxonomy
Classifies different types of implicit surfaces, including SDFs, occupancy fields, and level sets. Choose for understanding the broader context of implicit representations. [Implicit Surface Taxonomy](20200-implicit-surface-taxonomy.md)

## Operating characteristics
*   **Time Complexity:** Computing the SDF for a mesh-based object requires a nearest-point query for each voxel, which is $O(N \cdot M)$ for $N$ voxels and $M$ mesh primitives without acceleration structures. With spatial acceleration (e.g., BVH), this reduces significantly. Neural SDF evaluation is $O(1)$ per query but requires training time.
*   **Memory Complexity:** Dense grid SDFs require $O(N^3)$ memory for 3D resolution $N$. Sparse representations or neural networks reduce this to $O(N)$ or constant size, respectively.
*   **Strengths:** Provides metric distance information enabling robust sphere tracing; smooth gradients facilitate normal estimation; Boolean operations are simple and efficient.
*   **Limitations:** High memory usage for dense grids; difficult to represent non-watertight or open surfaces; neural SDFs require training and may suffer from artifacts if not regularized.
*   **When to Use:** Ideal for procedural geometry, sphere tracing, and applications requiring smooth surface interpolation or normal estimation.
*   **When Not to Use:** Not suitable for non-watertight surfaces or when exact mesh topology preservation is required without smoothing.

## Background
The Signed Distance Field was introduced as a continuous implicit representation mapping 3D points to signed distances, where the sign indicates inside/outside status and the amplitude represents distance from the watertight surface [S1]. It is widely used in non-linear registration techniques and shape skeletonization algorithms [S2]. In recent years, it has become a fundamental component in Implicit Neural Representations (INR) for 3D avatar modeling and human body reconstruction, often denoted as $f_\theta$ [S1].

## Sources
[S1] Human 3D Avatar Modeling with Implicit Neural Representation: A Brief Survey, https://doi.org/10.1109/icsps58776.2022.00148, fetched 2026-10-26
[S2] Virtual mastoidectomy performance evaluation through multi-volume analysis., https://doi.org/10.1007/s11548-012-0687-4, fetched 2026-11-08
