# Grid Structure and Topology


## Overview
This family defines the geometric and topological data structures used to represent the scalar field domain for isosurface extraction [G]. It solves the problem of mapping discrete volumetric data to a continuous mesh by establishing how grid vertices, cells, and boundaries are organized [G]. It sets itself apart from sibling methods by focusing on the underlying grid topology (structured, dual, or hierarchical) rather than the specific triangulation rules or ambiguity resolution strategies [G].

## Shared principles
Builds on [Marching Cubes Lookup and Configuration](2518-marching-cubes-lookup-and-configuration.md) for the precomputed lookup tables, case index calculation, and ambiguity handling. The family relies on a consistent vertex ordering and edge indexing scheme to map discrete scalar values to continuous surface geometry [G].

**Pseudocode**:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_case_index(values, isovalue):
    case_index = 0
    for i in range(8):
        if values[i] < isovalue:
            case_index |= (1 << i)
    return case_index

def interpolate_edge_intersection(v1, v2, p1, p2, isovalue):
    if abs(v2 - v1) < 1e-8:
        return (p1 + p2) / 2.0
    t = (isovalue - v1) / (v2 - v1)
    return p1 + t * (p2 - p1)

def extract_triangles(case_index, triangle_table, edge_intersections):
    triangles = []
    tri_indices = triangle_table[case_index]
    for i in range(16):
        if tri_indices[i] == -1:
            break
        e1 = tri_indices[i]
        e2 = tri_indices[i+1]
        e3 = tri_indices[i+2]
        p1 = edge_intersections[e1]
        p2 = edge_intersections[e2]
        p3 = edge_intersections[e3]
        triangles.append((p1, p2, p3))
    return triangles
# Note: edge_intersections must be precomputed by interpolating field values along the 12 cube edges [G].
```

## Subtopics

### [Marching Cubes on Structured Grids](6051-marching-cubes-on-structured-grids.md)
Extracts a 3D polygonal mesh from a discretized scalar field defined on a regular Cartesian grid. It sets itself apart by operating directly on the structured IJK grid topology using a fixed lookup table for cell triangulation, whereas methods like [Dual Contouring and Surface Nets](6319-dual-contouring-and-surface-nets.md) optimize vertex positions for accuracy, and [Marching Tetrahedra and Decomposition](11174-marching-tetrahedra-and-decomposition.md) decompose cells to resolve topological ambiguities. Choose it for standard medical imaging and neural field data where a regular grid is available [G].

### [Dual Grid Isosurface Extraction](10271-dual-grid-isosurface-extraction.md)
Extracts crack-free isosurfaces from Adaptive Mesh Refinement (AMR) data by constructing a dual grid where cell centers serve as vertices. It solves the "dangling node" problem inherent in standard Marching Cubes when applied to resampled vertex-centered AMR data, where mismatched interpolation at coarse-fine boundaries causes topological discontinuities. Unlike sibling methods that fix cracks by post-processing polygons or collapsing polylines, this approach eliminates dangling nodes structurally by leveraging the geometric gap between coarse and fine dual grids. Choose it when working with AMR datasets to ensure manifold continuity across refinement levels [G].

### [Dual-Grid Non-Manifold Contouring](14936-dual-grid-non-manifold-contouring.md)
Extracts non-manifold meshes from segmented volumes containing multiple materials by utilizing a dual grid structure where material attributes are associated with voxel centers rather than cell corners. Unlike generalized marching cubes, which requires large lookup tables for multi-material cases, this method avoids lookup tables entirely by generating quadrilateral faces at voxel boundaries and resolving topological ambiguities through a priority-based vertex splitting mechanism. Choose it for multi-material segmentation where non-manifold connectivity is required [G].

### [Hypertree Grid Isocontouring](18933-hypertree-grid-isocontouring.md)
Uses a hypertree data structure to represent the grid topology, allowing for efficient traversal and isosurface extraction on hierarchical data. Choose it when memory efficiency and adaptive traversal are critical for large-scale volumetric datasets [G].

### [Batched Marching Cubes](19943-batched-marching-cubes.md)
Processes multiple voxels or grid regions in parallel batches to optimize memory access patterns and throughput on modern hardware. Choose it for high-performance rendering pipelines where latency and throughput are prioritized over per-voxel precision [G].

### [Sparse and Hierarchical Grid Methods](21072-sparse-and-hierarchical-grid-methods.md)
Adapts the grid structure to store only non-empty or active regions, reducing memory footprint and computation time for sparse volumetric data. Choose it when the scalar field is highly sparse or when dynamic grid updates are necessary [G].

### [GPU Acceleration and Parallel Pipelines](4136-gpu-acceleration-and-parallel-pipelines.md)
Maps the grid traversal and triangle generation steps to GPU parallel execution models, leveraging shared memory and coalesced memory access. Choose it for real-time applications requiring massive parallelism and low CPU overhead [G].

### [Surface-following Marching Cubes with Blocking](8966-surface-following-marching-cubes-with-blocking.md)
Optimizes traversal by following the surface across grid cells, reducing the number of empty voxels processed. Choose it when the isosurface is localized or when processing large volumes with significant empty space [G].

### [Real-time particle isosurface extraction](10275-real-time-particle-isosurface-extraction.md)
Adapts the grid structure to handle particle-based scalar fields, often using kernel density estimation on a grid. Choose it for fluid dynamics or particle simulations where the scalar field is derived from discrete particles [G].

### [Lookup Table and Ambiguity Handling](10327-lookup-table-and-ambiguity-handling.md)
Focuses on the generation and modification of the `edgeTable` and `triangleTable` to resolve topological ambiguities and ensure manifold output. Choose it when standard LUTs produce cracks or non-manifold artifacts in ambiguous cases [G].

### [Algorithmic Modifications and Improvements](11203-algorithmic-modifications-and-improvements.md)
Covers enhancements to the base grid topology processing, such as adaptive grid refinement and error minimization. Choose it when higher geometric accuracy or adaptive resolution is required [G].

### [Fiber Surface Extraction](14192-fiber-surface-extraction.md)
Modifies the grid topology to extract tubular structures, often using a directional grid or anisotropic interpolation. Choose it for diffusion tensor imaging or fiber tractography where anisotropy is dominant [G].

### [Mesh Post-processing and Simplification](14742-mesh-post-processing-and-simplification.md)
Applies topological simplification and cleaning to the extracted mesh, often adjusting vertex positions on the grid structure. Choose it when reducing polygon count or fixing minor topological defects is necessary [G].

### [Analytical Error Approximation for Edge-Crossing](15100-analytical-error-approximation-for-edge-crossing.md)
Refines the edge intersection calculation using analytical error bounds to improve geometric accuracy on the grid. Choose it when sub-voxel precision is critical for the application [G].

## Background
Not given in the sources.

## Sources
[S1] Marching cubes technique for volumetric visualization accelerated with graphics processing units, https://doi.org/10.1007/s13173-012-0097-z, fetched 2026-10-26
