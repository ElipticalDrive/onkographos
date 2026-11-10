# Marching Cubes with Octree


## Overview
Marching Cubes with Octree is an isosurface extraction method that applies the Marching Cubes algorithm over an octree data structure to convert volumetric scalar fields into triangle meshes [S1]. It solves the problem of efficiently processing sparse or unbounded datasets by replacing fixed-grid storage with a hierarchical structure, avoiding the iteration overhead of dense arrays [S1]. Unlike standard grid-based marching cubes, it adapts the domain traversal to the octree topology, though it requires uniform node sizes around the isosurface to prevent topological cracks [S1].

## How it works
The method operates on a scalar function defined over a 3D bounding box, using an octree to discretize the domain and a domain object to interface with the extraction engine.

**Inputs and Outputs:**
*   **Input:** A 3D bounding box (`bbox`), a split predicate function (`split_predicate`) to refine the octree, a scalar function (`function`) evaluating values at points, and an isovalue threshold.
*   **Output:** A set of vertices (`Point_range`) and a set of triangles (`Polygon_range`).

**Data Structures:**
1.  **Octree:** Initialized with bounding box points. Nodes are refined based on the split predicate. The structure stores the hierarchical subdivision of the domain.
2.  **Values:** An object containing the scalar function and a reference to the octree, allowing value queries at arbitrary points within the octree's bounds.
3.  **MC_Domain:** A domain object wrapping the octree and values. It provides the interface required by the marching cubes algorithm to traverse cells and query scalar values.

**Steps:**
1.  Define the bounding box points from `bbox.xmin/ymin/zmin` and `bbox.xmax/ymax/zmax` [S1].
2.  Construct an `Octree` object initialized with these bounding box points [S1].
3.  Refine the octree by calling `octree.refine(split_predicate)`. The predicate determines which nodes are subdivided based on the scalar field characteristics [S1].
4.  Create a `Values` object containing the scalar function and the constructed octree [S1].
5.  Initialize `Point_range` and `Polygon_range` containers to store the output vertices and triangles, respectively [S1].
6.  Construct an `MC_Domain` object passing the octree and values [S1].
7.  Call `CGAL::Isosurfacing::marching_cubes` with the `MC_Domain`, the isovalue, and the output ranges. The implementation supports parallel execution via `Parallel_if_available_tag` [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def run_marching_cubes_octree(bbox, split_predicate, scalar_function, isovalue):
    # 1. Define bounding box points
    bbox_points = [
        Point(bbox.xmin, bbox.ymin, bbox.zmin),
        Point(bbox.xmax, bbox.ymax, bbox.zmax)
    ]
    
    # 2. Construct Octree
    octree = Octree(bbox_points)
    
    # 3. Refine octree using split predicate
    octree.refine(split_predicate)
    
    # 4. Create Values object
    values = Values(scalar_function, octree)
    
    # 5. Initialize output containers
    points = Point_range()
    triangles = Polygon_range()
    
    # 6. Construct MC_Domain
    mcdomain = MC_Domain(octree, values)
    
    # 7. Execute Marching Cubes
    # Note: Parallel_if_available_tag is a tag for the CGAL library
    marching_cubes(mcdomain, isovalue, points, triangles, Parallel_if_available_tag())
    
    return points, triangles
```

**Parameters:**
*   **isovalue:** The scalar threshold for surface extraction. Typical value: `0.3` (example from source) [S1].
*   **split_predicate:** A function determining octree refinement. Common variants include `Refine_one_eighth` (uniform subdivision) or `Refine_around_isovalue` (adaptive based on scalar value proximity) [S1].

**Edge Cases and Pitfalls:**
*   **Cracks:** Using an octree with varying depths around the isosurface produces cracks at regions where octree levels are discontinuous [S1]. The method behaves correctly only if nodes are split with uniform size around the surface, resulting in a manifold surface [S1].
*   **Discontinuities:** If adjacent voxels belong to different octree levels, the density field is not continuous, leading to disconnected triangles [S2].

## Subtopics

### [Linear Hashed Marching Cubes](1294-linear-hashed-marching-cubes.md)
Linear Hashed Marching Cubes is a surface extraction method that converts volumetric isosurfaces into triangle meshes by applying the Marching Cubes algorithm over a linear hashed octree data structure. It serves as a loose implementation of the "Fast Generation of Pointerless Octree Duals" technique, designed to accelerate mesh generation compared to standard grid-based marching cubes. By replacing fixed-grid storage with a linear hashed octree, the method efficiently handles sparse or unbounded datasets without the iteration overhead of dense arrays or the query latency of traditional pointer-based octrees.

### [Adaptive Resolution Isosurface Rendering with Two-Pass Discontinuity Handling](1340-adaptive-resolution-isosurface-rendering-with-two-pass-disco.md)
This method extracts isosurfaces from adaptive resolution (AR) octree volume data by decoupling geometry generation from discontinuity resolution. It solves the topological inconsistencies that arise when fine-resolution cubes share edges or faces with coarser, unsubdivided neighbors. Unlike uniform resolution methods, it reduces data access and processing time by skipping large uniform regions, while the two-pass approach ensures a watertight mesh without requiring complex on-the-fly edge matching logic during the primary traversal.

### [Adaptive Marching Cubes](1351-adaptive-marching-cubes.md)
Adaptive Marching Cubes is an extension of the standard Marching Cubes algorithm designed to extract isosurfaces from Adaptive Resolution (AR) octree data. It solves the problem of topological inconsistencies and geometric artifacts that arise when adjacent cubes in the mesh have different resolutions. Unlike uniform marching cubes, it handles missing vertices and discontinuities by deferring the processing of coarser nodes that conflict with finer neighbors, ensuring a watertight mesh across resolution boundaries.

### [DMC Octree Contour Extraction](3661-dmc-octree-contour-extraction.md)
DMC Octree Contour Extraction is an adaptive isosurface extraction method that generates a manifold triangle mesh from volumetric scalar field data using a dual-contouring approach on an octree structure. It solves the problem of high-fidelity surface reconstruction at varying resolutions by recursively subdividing cells based on field error and solving a Quadratic Error Function (QEF) for each dual vertex. It sets itself apart from its sibling [Linear Hashed Marching Cubes](1294-linear-hashed-marching-cubes.md) by using dual vertices and QEF solving rather than linear interpolation on grid edges, and from [Adaptive Marching Cubes](1351-adaptive-marching-cubes.md) by explicitly managing topological consistency through a recursive face/edge/vertex processing pipeline that ensures shared boundaries between adjacent octree nodes are handled deterministically.

### [Simplicial Partition Isosurfacing](9787-simplicial-partition-isosurfacing.md)
Simplicial Partition Isosurfacing extracts a manifold, intersection-free isosurface from a scalar function defined over an arbitrary octree by partitioning space into simplices (tetrahedra) dual to minimal octree cells. Unlike Dual Contouring, which produces non-manifold geometry in ambiguous sign configurations, and Dual Marching Cubes, which can generate self-intersecting surfaces due to non-convex dual cells, this method guarantees manifold topology and prevents tetrahedron inversion, ensuring the surface cannot self-intersect.

### Sparse-Voxel TSDF Fusion and Marching Cubes
This variant extracts a mesh by adapting TSDF-Fusion and Marching Cubes into a sparse voxel representation [S2]. It implements sparse-voxel TSDF-Fusion to compute truncated signed distance values of the sparse grid points, then extracts the surface of the zero-level set using sparse-voxel Marching Cubes [S2]. Duplicated vertices from adjacent voxels are merged to produce a unique set of vertices [S2]. If adjacent voxels belong to different octree levels, all voxels are subdivided to the finest level to remove discontinuities in the density field [S2]. Unlike methods relying on coarse geometry priors, this method models the scene only with a density field [S2]. Pitfalls include producing unnecessary geometric bumps for texture details and generating disconnected triangles when adjacent voxels are at different octree levels due to density field discontinuity [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the octree refinement depth and the number of leaf nodes intersecting the isosurface. Parallel execution is supported via `Parallel_if_available_tag` [S1].
*   **Memory Complexity:** Proportional to the number of octree nodes. The linear hashed variant reduces overhead compared to pointer-based octrees [S1].
*   **Strengths:** Efficient handling of sparse or unbounded datasets; avoids dense array iteration overhead [S1].
*   **Limitations:** Produces cracks if octree nodes have varying depths around the isosurface; requires uniform node sizes for manifold surfaces [S1]. Adaptive variants may produce geometric bumps for texture details [S2].
*   **When to use:** When the volumetric data is sparse or unbounded, and uniform refinement around the isosurface is feasible or enforced.
*   **When not to use:** When the octree structure has significant depth variations near the surface, as this leads to topological cracks [S1].

## Background
The method is part of the CGAL Isosurfacing library, specifically `CGAL::Isosurfacing::marching_cubes` with an `MC_Domain` adapter for octree structures [S1]. The underlying Marching Cubes algorithm was introduced in 1987 [S2]. The octree-based approach builds on the need to handle sparse data efficiently, with variants like Linear Hashed Marching Cubes implementing techniques from "Fast Generation of Pointerless Octree Duals" [S1]. Sparse-voxel variants cite TSDF-Fusion (KinectFusion, 2011) and Marching Cubes (1987) as foundational references [S2].

## Sources
[S1] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
[S2] Sparse Voxels Rasterization: Real-time High-fidelity Radiance Field Rendering, https://doi.org/10.1109/cvpr52734.2025.01509, fetched 2026-10-26
