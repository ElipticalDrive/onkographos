# Marching Cubes with Octree


## Overview
Marching Cubes with Octree extracts a triangle mesh from an implicit scalar field defined over an adaptive octree structure, replacing the uniform Cartesian grid of standard Marching Cubes [S1]. It sets itself apart from its siblings by relying on a generic domain interface (`MC_Domain`) that decouples the traversal logic from the specific octree implementation, allowing for parallel execution and arbitrary split predicates [S1]. Unlike uniform grid methods, it adapts resolution to the field's complexity, but it lacks inherent guarantees against topological cracks if the octree depth varies discontinuously around the isosurface [S1].

## How it works
The method operates by constructing an octree based on a split predicate, defining a domain that links the octree to a scalar function, and invoking the marching cubes algorithm which traverses the octree to generate triangles [S1].

**Inputs:**
*   `bbox`: A 3D bounding box defined by minimum and maximum coordinates $(x_{min}, y_{min}, z_{min})$ and $(x_{max}, y_{max}, z_{max})$ [S1].
*   `split_predicate`: A function or object that determines whether an octree node should be subdivided, typically based on the scalar field's variation or gradient [S1].
*   `scalar_function`: A function $f: \mathbb{R}^3 \rightarrow \mathbb{R}$ that evaluates the implicit field at any point [S1].
*   `isovalue`: The scalar threshold $I$ at which the isosurface is extracted [S1].

**Outputs:**
*   `points`: A container of 3D vertices forming the mesh [S1].
*   `triangles`: A container of triangle indices referencing the vertices [S1].

**Data Structures:**
1.  **Octree**: A hierarchical spatial partitioning structure. Each node represents a cubic volume. The root covers the `bbox`. Nodes are refined recursively if `split_predicate` returns true. The structure supports traversal and neighbor access [S1].
2.  **Values**: An object holding the `scalar_function` and a reference to the `octree`. It provides the interface for the domain to query field values at specific locations, potentially leveraging the octree's hierarchy for efficient evaluation [S1].
3.  **MC_Domain**: An abstraction layer that implements the interface expected by the marching cubes algorithm. It maps the algorithm's requests for cell vertices and edge intersections to the underlying octree structure and scalar function [S1].

**Algorithm Steps:**

1.  **Initialize Bounding Box**: Define the root cube of the octree using the input `bbox` points [S1].
2.  **Construct Octree**: Create an `Octree` object initialized with the bounding box points [S1].
3.  **Refine Octree**: Call `octree.refine(split_predicate)` to recursively subdivide nodes. The predicate determines refinement based on the scalar field's behavior within the node [S1].
4.  **Create Values Object**: Instantiate a `Values` object containing the `scalar_function` and the constructed `octree` [S1].
5.  **Initialize Output Containers**: Create empty containers `Point_range points` and `Polygon_range triangles` to store the resulting vertices and triangles [S1].
6.  **Construct Domain**: Instantiate an `MC_Domain` object, passing the `octree` and `values` objects. This binds the geometric structure and the scalar field to the marching cubes interface [S1].
7.  **Execute Marching Cubes**: Call `CGAL::Isosurfacing::marching_cubes` with the `MC_Domain`, `isovalue`, and output ranges. The algorithm uses `Parallel_if_available_tag` to enable parallel processing if supported by the environment [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode representing the CGAL API usage described in [S1]
# Note: This is a high-level representation of the C++ template usage.
# Actual implementation requires CGAL headers and types.

function run_marching_cubes_octree(bbox, split_predicate, scalar_function, isovalue):
    # 1. Define bounding box points
    bbox_points = [
        Point(bbox.xmin, bbox.ymin, bbox.zmin),
        Point(bbox.xmax, bbox.ymax, bbox.zmax)
    ]
    
    # 2. Construct Octree
    octree = Octree(bbox_points)
    
    # 3. Refine the octree using the split predicate
    # The predicate determines if a node needs subdivision
    octree.refine(split_predicate)
    
    # 4. Create Values object linking function and octree
    values = Values(scalar_function, octree)
    
    # 5. Initialize output containers
    points = [] # List of 3D points
    triangles = [] # List of triangle indices
    
    # 6. Construct MC_Domain
    mcdomain = MC_Domain(octree, values)
    
    # 7. Call marching cubes
    # The algorithm traverses the octree, evaluates the scalar function,
    # and generates triangles into 'points' and 'triangles'.
    # Parallel_if_available_tag enables multi-threading.
    marching_cubes(mcdomain, isovalue, points, triangles, Parallel_if_available_tag)
    
    return points, triangles
```

**Parameters:**
*   `isovalue`: The threshold value. Typical values depend on the scalar field's range. In the CGAL example, a default of `0.3` is used for a normalized sphere function [S1].
*   `split_predicate`: Defines the adaptive resolution. Common predicates include `Refine_around_isovalue` which splits nodes where the scalar field crosses the isovalue, or fixed-depth splitters like `Refine_one_eighth` [S1].

**Edge Cases and Pitfalls:**
*   **Cracks**: Using an octree with varying depths around the isosurface produces cracks at regions where octree levels are discontinuous [S1]. The standard Marching Cubes algorithm assumes a uniform grid topology; when adjacent cells have different resolutions, shared edges may have mismatched intersection points or missing vertices [S1].
*   **Manifold Guarantee**: The octree behaves well and produces a manifold surface only if nodes are split with a uniform size around the surface [S1].
*   **Precision**: The output precision is controlled by the floating-point type used for the scalar function and coordinates. In the CGAL example, `17` decimal digits are used for polylines output [S1].

## Subtopics

### [Linear Hashed Marching Cubes](1294-linear-hashed-marching-cubes.md)
Linear Hashed Marching Cubes extracts triangle meshes from implicit scalar fields by applying the Marching Cubes algorithm over a linear hashed octree. It replaces the standard grid-based traversal with a pointerless octree structure to reduce memory overhead and improve performance. It builds on [Marching Cubes with Octree](1259-marching-cubes-with-octree.md) but uses a linear hash table for node storage instead of a primal grid.

### [Adaptive Resolution Isosurface Rendering with Two-Pass Discontinuity Handling](1340-adaptive-resolution-isosurface-rendering-with-two-pass-disco.md)
This variant extends Marching Cubes to handle isosurface rendering on Adaptive Resolution (AR) octree data, addressing inconsistencies between cells of different resolutions. It uses a two-pass approach: a first pass generates triangles using standard MC, while a second pass processes nodes with missing vertices. If a coarser node finds two vertices on the same edge in its fine edge vertex list, it subdivides the cube. It sets itself apart from [Linear Hashed Marching Cubes](1294-linear-hashed-marching-cubes.md) by explicitly handling resolution mismatches through subdivision and post-processing, rather than relying on a flat hash structure. It is chosen when the octree has arbitrary depth variations that standard MC cannot handle without cracks.

### [DMC Octree Contour Extraction](3661-dmc-octree-contour-extraction.md)
DMC Octree Contour Extraction is an adaptive isosurface extraction method that traverses a pre-built octree to generate a triangle mesh via dual-vertex construction and contour processing. It solves the problem of extracting high-quality surfaces from volumetric data with varying density by recursively subdividing cells based on field error metrics. It sets itself apart from its sibling [Linear Hashed Marching Cubes](1294-linear-hashed-marching-cubes.md) by using a hierarchical octree structure with explicit contour propagation across shared faces and edges, rather than a flat hash-based lookup, and from [Adaptive Resolution Isosurface Rendering with Two-Pass Discontinuity Handling](1340-adaptive-resolution-isosurface-rendering-with-two-pass-disco.md) by relying on a single-pass recursive traversal with dual-vertex QEF solving rather than a two-pass discontinuity-aware approach.

## Operating characteristics
*   **Time Complexity**: Depends on the number of octree nodes and the complexity of the split predicate. The marching cubes traversal is linear with respect to the number of leaf nodes in the octree [S1]. Parallel execution can reduce wall-clock time on multi-core systems [S1].
*   **Memory Complexity**: Proportional to the number of octree nodes. The linear hashed variant reduces overhead by using a hash table instead of pointers [S1].
*   **Measured Performance**: Evaluated on a Dell PC with a 450 MHz Pentium microprocessor and 256 Mbytes of RAM running Red Hat Linux 6.1, using Java 3D for the AR variant [S2, unofficial]. Specific performance metrics for the CGAL implementation are not provided in the sources [S1].
*   **Strengths**: Adaptive resolution allows for high detail where needed and coarse representation elsewhere, saving memory and computation. The domain-based design allows flexibility in octree implementation [S1].
*   **Limitations**: Cracks can occur if the octree is not adapted uniformly around the surface. The algorithm does not guarantee manifoldness without specific splitting strategies [S1].
*   **When to use**: When the scalar field has varying complexity and a uniform grid would be too expensive or wasteful. When the octree can be constructed such that depth changes are gradual or handled by a variant like Adaptive Marching Cubes.
*   **When not to use**: When strict manifoldness is required and the octree structure cannot be controlled to avoid depth discontinuities. When the scalar field is uniform and a simple grid-based MC is sufficient.

## Background
The method is part of the CGAL (Computational Geometry Algorithms Library) Isosurfacing module [S1]. It was developed to provide a flexible interface for isosurfacing on various spatial data structures, including octrees [S1]. The underlying Marching Cubes algorithm was originally introduced by Lorensen and Cline [S1, implied by standard knowledge, but source [S1] cites CGAL implementation]. The specific integration with octrees and the `MC_Domain` interface is documented in the CGAL reference manual [S1]. The Adaptive Resolution variant was proposed by [S2, unofficial] to handle inconsistencies in multiresolution data. The sparse-voxel TSDF fusion approach using Marching Cubes was introduced in [S3] for real-time radiance field rendering.

## Sources
[S1] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
[S2] IsoSurface Rendering of Adaptive Resolution Data, https://www.cs.unh.edu/~rdb/papers/ARiso.pdf, fetched 2026-10-29 (unofficial)
[S3] Sparse Voxels Rasterization: Real-time High-fidelity Radiance Field Rendering, https://doi.org/10.1109/cvpr52734.2025.01509, fetched 2026-10-26
