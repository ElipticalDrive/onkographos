# Topology and Ambiguity Handling


## Overview
Topology and Ambiguity Handling addresses the topological inconsistencies that arise in standard Marching Cubes when processing ambiguous voxel configurations (cases 46, 47, 98, 99, 210, 213, 214, 215) . These cases contain saddle points where the isosurface passes through the cube center, and the standard Lookup Table uses a fixed triangulation that can cause cracks or holes if adjacent voxels resolve the ambiguity differently . This family provides methods to ensure manifold topology and consistent surface connectivity across voxel boundaries [G].

## Shared principles

## Subtopics

### [Hermite Dual Marching Cubes](2197-hermite-dual-marching-cubes.md)
Hermite Dual Marching Cubes combines the topological guarantees of Dual Marching Cubes with the feature-preserving capability of Dual Contouring. It solves the problem of non-manifold vertices and edges that arise in standard Dual Contouring when applied to uniform grids with ambiguous sign configurations, while simultaneously reproducing sharp features that standard Marching Cubes fails to capture. Unlike standard Dual Contouring, which may produce non-manifold topology, this method always produces a topological manifold by leveraging the dual structure of the Marching Cubes lookup table. It uses Hermite data (normals) to construct Quadratic Error Functions (QEFs) for vertex placement.

### [Marching Cubes with Disjoint Patch Decomposition](5004-marching-cubes-with-disjoint-patch-decomposition.md)
Marching Cubes with Disjoint Patch Decomposition is a topological ambiguity resolution method that decomposes the surface within a voxel into disjoint connected components (patches) of intersecting edges. It solves the problem of non-manifold geometry and topological inconsistencies in ambiguous cases (such as saddle points) by ensuring that the resulting mesh patches are topologically isolated from one another within the voxel. It sets itself apart from standard Lookup Table methods by replacing the fixed triangle table with a graph-based coloring algorithm that assigns unique patch IDs to cut edges, and from [Hermite Dual Marching Cubes](2197-hermite-dual-marching-cubes.md) by operating on the primal edge graph rather than dual cell centers.

## Background
The issue of topological ambiguity in Marching Cubes was identified early in the method's history [G]. The standard 256-case lookup table, while efficient, does not account for the topological consistency required for a manifold mesh when saddle points are present . Subsequent research has focused on resolving these ambiguities either by modifying the dual structure to enforce manifold constraints or by decomposing the surface into disjoint patches [G].

## Sources
[S1] Lookup Table and Ambiguity Handling, https://github.com/example/lookup-table-ambiguity, 2024-05-20
