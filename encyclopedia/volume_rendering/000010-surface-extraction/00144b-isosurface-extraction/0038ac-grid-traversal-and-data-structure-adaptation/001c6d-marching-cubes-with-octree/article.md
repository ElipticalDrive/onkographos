# Marching Cubes with Octree


## Overview
Marching Cubes with Octree extracts a triangle mesh isosurface from a scalar field defined over an adaptive octree structure, enabling efficient processing of sparse or highly non-uniform volumetric data. Unlike uniform grid methods, it adapts resolution to the geometry, but it requires careful handling of depth discontinuities to avoid topological cracks. It differs from [Sparse Marching Cubes](3825-sparse-marching-cubes.md) by using a structured octree refinement predicate rather than a purely sparse hash map, and from [Hierarchical Marching Cubes Enumeration](5223-hierarchical-marching-cubes-enumeration.md) by relying on a domain object that abstracts the traversal logic for the CGAL isosurfacing library.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D bounding box (`bbox`), a scalar field function $f(\mathbf{p})$ returning a value for any point $\mathbf{p}$, an isovalue $I$, and a split predicate (refinement function) that determines octree node subdivision.
*   **Output:** A set of vertices (points) and a set of triangles (polygons) representing the isosurface $f(\mathbf{p}) = I$.

**Data Structures:**
1.  **Octree:** A hierarchical spatial partitioning structure. Each node represents a cubic voxel. Nodes are refined based on the `split_predicate`. The structure supports traversal and querying of the scalar field at leaf nodes.
2.  **Values:** An object encapsulating the scalar function $f$ and the reference to the Octree. It provides the interface for the domain to evaluate field values.
3.  **MC_Domain:** A domain object required by the CGAL `Isosurfacing` module. It wraps the Octree and Values, providing the necessary iterators and geometric primitives for the Marching Cubes algorithm to traverse the octree leaves and generate mesh elements.
4.  **Point_range / Polygon_range:** Containers (typically `std::vector`) storing the output vertices and triangle indices.

**Algorithm Steps:**
1.  **Initialize Bounding Box:** Define the 3D bounding box using minimum and maximum coordinates: $\mathbf{p}_{min} = (x_{min}, y_{min}, z_{min})$ and $\mathbf{p}_{max} = (x_{max}, y_{max}, z_{max})$.
2.  **Construct Octree:** Create an `Octree` object initialized with $\mathbf{p}_{min}$ and $\mathbf{p}_{max}$.
3.  **Refine Octree:** Apply the `split_predicate` function to recursively refine the octree. The predicate determines if a node should be subdivided into 8 children based on the scalar field behavior within that voxel (e.g., if the isosurface passes through it).
4.  **Create Values Object:** Instantiate a `Values` object containing the scalar function $f$ and the refined `Octree`.
5.  **Initialize Output Containers:** Create empty containers `points` (for vertices) and `triangles` (for triangle indices).
6.  **Construct MC_Domain:** Instantiate `MC_Domain` passing the `Octree` and `Values` objects. This domain object abstracts the traversal of the octree leaves.
7.  **Execute Marching Cubes:** Call `CGAL::Isosurfacing::marching_cubes` with the `MC_Domain`, the `isovalue` $I$, and the output containers. The algorithm may use parallel execution if available.
8.  **Post-processing (Optional):** The octree can be dumped to polylines or the mesh can be written to a file format (e.g., OFF).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def run_marching_cubes_octree(bbox_min, bbox_max, scalar_function, isovalue, split_predicate):
    """
    Extracts an isosurface from a scalar field using an adaptive octree.
    
    Args:
        bbox_min: Tuple (x, y, z) of minimum bounding box coordinates.
        bbox_max: Tuple (x, y, z) of maximum bounding box coordinates.
        scalar_function: Function f(p) -> float, evaluating the scalar field at point p.
        isovalue: Float, the threshold value for the isosurface.
        split_predicate: Function node -> bool, returns True if the node should be subdivided.
        
    Returns:
        points: List of 3D vertices.
        triangles: List of triangle indices (triplets of vertex indices).
    """
    # Step 1: Define bounding box points
    bbox_points = [bbox_min, bbox_max]
    
    # Step 2: Construct Octree
    octree = Octree(bbox_points)
    
    # Step 3: Refine the octree using the split predicate
    # The predicate is applied recursively to determine leaf nodes
    octree.refine(split_predicate)
    
    # Step 4: Create Values object
    values = Values(scalar_function, octree)
    
    # Step 5: Initialize output containers
    points = []
    triangles = []
    
    # Step 6: Construct MC_Domain
    # This domain object provides the interface for the marching cubes algorithm
    # to traverse the octree and access field values
    mcdomain = MC_Domain(octree, values)
    
    # Step 7: Call Marching Cubes
    # The algorithm traverses the octree leaves, identifies intersections with the isovalue,
    # and generates triangles. Parallel execution is used if supported.
    marching_cubes(mcdomain, isovalue, points, triangles, parallel_tag=Parallel_if_available_tag)
    
    return points, triangles
```

**Parameters:**
*   `isovalue`: The scalar value defining the surface. Typical values depend on the field normalization (e.g., 0.0 for zero-level set, 0.3 for normalized density).
*   `split_predicate`: Defines the adaptive resolution. A common strategy is `Refine_around_isovalue`, which subdivides nodes where the scalar field crosses the isovalue. Another is `Refine_one_eighth`, which enforces a fixed depth ratio.

**Edge Cases and Pitfalls:**
*   **Cracks at Depth Discontinuities:** If the octree has varying depths around the isosurface (i.e., a fine node is adjacent to a coarse node), standard Marching Cubes will produce cracks or non-manifold geometry because the coarse node's single triangle does not align with the fine node's subdivided triangles [S1].
*   **Manifold Guarantee:** The method produces a manifold surface only if nodes are split with uniform size around the surface [S1].
*   **Precision:** The output precision is controlled by the floating-point type used for coordinates (e.g., `double` in CGAL).

## Subtopics

### [Linear Hashed Marching Cubes](1294-linear-hashed-marching-cubes.md)
Linear Hashed Marching Cubes extracts triangle meshes from implicit scalar fields by applying the Marching Cubes algorithm over a linear hashed octree. It replaces the standard grid-based traversal with a pointerless octree structure to reduce memory overhead and improve performance. It builds on [Marching Cubes with Octree](1259-marching-cubes-with-octree.md) but uses a linear hash table for node storage instead of a primal grid.

### [Adaptive Resolution Isosurface Rendering with Two-Pass Discontinuity Handling](1340-adaptive-resolution-isosurface-rendering-with-two-pass-disco.md)
This method extracts isosurfaces from adaptive resolution (AR) octree volume data by decoupling geometry generation from discontinuity resolution. It solves the topological inconsistencies that arise when fine-resolution cubes share edges or faces with coarser, unsubdivided neighbors. Unlike uniform resolution methods, it reduces data access and processing time by skipping large uniform regions, while the two-pass approach ensures a watertight mesh without requiring complex on-the-fly edge matching logic during the primary traversal.

### [Adaptive Marching Cubes](1351-adaptive-marching-cubes.md)
Adaptive Marching Cubes is an extension of the standard Marching Cubes algorithm designed to extract isosurfaces from Adaptive Resolution (AR) octree data. It solves the problem of topological inconsistencies and geometric artifacts that arise when adjacent cubes in the mesh have different resolutions. Unlike uniform marching cubes, it handles missing vertices and discontinuities by deferring the processing of coarser nodes that conflict with finer neighbors, ensuring a watertight mesh across resolution boundaries.

### [DMC Octree Contour Extraction](3661-dmc-octree-contour-extraction.md)
DMC Octree Contour Extraction is an adaptive isosurface extraction method that generates a manifold triangle mesh from volumetric scalar field data using a dual-contouring approach on an octree structure. It solves the problem of high-fidelity surface reconstruction at varying resolutions by recursively subdividing cells based on field error and solving a Quadratic Error Function (QEF) for each dual vertex. It sets itself apart from its sibling [Linear Hashed Marching Cubes](1294-linear-hashed-marching-cubes.md) by using dual vertices and QEF solving rather than linear interpolation on grid edges, and from [Adaptive Marching Cubes](1351-adaptive-marching-cubes.md) by explicitly managing topological consistency through a recursive face/edge/vertex processing pipeline that ensures shared boundaries between adjacent octree nodes are handled deterministically.

### Sparse-Voxel TSDF Fusion and Marching Cubes
This variant adapts TSDF-Fusion and Marching Cubes into a sparse voxel representation. It computes truncated signed distance values for sparse grid points and extracts the zero-level set surface. Duplicated vertices from adjacent voxels are merged. If adjacent voxels belong to different octree levels, all voxels are subdivided to the finest level to remove discontinuities in the density field [S2]. Unlike methods relying on coarse geometry priors, this method models the scene only with a density field [S2]. It tends to produce unnecessary geometric bumps for texture details [S2]. When adjacent voxels belong to different octree levels, the extracted triangle may not be connected because the density field is not continuous for voxels in different levels [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the number of leaf nodes in the octree and the complexity of the split predicate. The marching cubes traversal is linear in the number of leaves.
*   **Memory Complexity:** Proportional to the number of octree nodes. The octree structure is more memory-efficient than a uniform grid for sparse fields.
*   **Strengths:** Efficient handling of sparse data; adaptive resolution reduces computation in empty or uniform regions.
*   **Limitations:** Prone to cracks if depth discontinuities are not handled (e.g., via uniform splitting or post-processing subdivision) [S1].
*   **When to Use:** When the scalar field is sparse or has high-frequency details in limited regions, making uniform grids inefficient.
*   **When Not to Use:** When the field is dense and uniform, or when strict manifold guarantees are required without additional post-processing to handle depth discontinuities.

## Background
The method is implemented in the CGAL library as `CGAL::Isosurfacing::marching_cubes` with an octree domain. The underlying Marching Cubes algorithm was introduced by Lorensen and Cline in 1987 [S2]. The integration with octrees allows for adaptive sampling, addressing the limitations of uniform grids for large-scale or sparse volumetric data. The specific implementation details, including the `MC_Domain` interface and split predicates like `Refine_around_isovalue`, are documented in the CGAL Isosurfacing 3 module [S1].

## Sources
[S1] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
[S2] Sparse Voxels Rasterization: Real-time High-fidelity Radiance Field Rendering, https://doi.org/10.1109/cvpr52734.2025.01509, fetched 2026-10-26
