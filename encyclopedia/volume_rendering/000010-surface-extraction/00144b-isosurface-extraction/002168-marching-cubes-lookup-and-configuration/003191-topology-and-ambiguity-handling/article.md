# Topology and Ambiguity Handling

Also called: Ambiguity Resolution, Topological Correctness

## Overview
Topology and Ambiguity Handling addresses the topological errors inherent in standard Marching Cubes, specifically non-manifold vertices and edges that arise in ambiguous sign configurations (such as saddle points) [S2, unofficial]. It ensures the output mesh is a topological manifold by dynamically resolving connectivity within ambiguous cells, distinguishing it from static lookup table approaches [2518-marching-cubes-lookup-and-configuration.md].

## Shared principles
The family shares the goal of producing a manifold mesh from scalar field data by resolving topological ambiguities that standard lookup tables cannot handle deterministically.

1.  **Ambiguity Identification**: The methods identify cell configurations where the sign pattern of the 8 vertices does not uniquely determine the surface topology (e.g., the "saddle" cases) [S2, unofficial].
2.  **Graph-Based Connectivity**: Instead of relying on a pre-computed static lookup table, these methods analyze the connectivity of edges within the cube. This often involves constructing an adjacency graph of edges or tetrahedra [5004-marching-cubes-with-disjoint-patch-decomposition.md].
3.  **Component Decomposition**: Ambiguous regions are decomposed into disjoint connected components or patches. This ensures that cut edges are grouped into distinct, non-intersecting surfaces, preventing self-intersections or non-manifold joints [5004-marching-cubes-with-disjoint-patch-decomposition.md].
4.  **Manifold Guarantee**: The primary constraint is the production of a topological manifold, where every edge is shared by exactly two faces and every vertex has a neighborhood homeomorphic to a disk [S1][S2, unofficial].

## Subtopics

### [Hermite Dual Marching Cubes](2197-hermite-dual-marching-cubes.md)
Combines Dual Marching Cubes topology with Dual Contouring features. It uses Hermite data (points and normals) to construct Quadratic Error Functions (QEF) for vertex placement, ensuring manifold topology while preserving sharp features.

### [Marching Cubes with Disjoint Patch Decomposition](5004-marching-cubes-with-disjoint-patch-decomposition.md)
Resolves ambiguous cases by decomposing the surface into disjoint connected components of edges using graph-based coloring. It replaces the static lookup table with dynamic graph analysis to ensure non-intersecting patches [5004-marching-cubes-with-disjoint-patch-decomposition.md].

## Background
Standard Marching Cubes [2518-marching-cubes-lookup-and-configuration.md] uses a static lookup table to resolve the 256 possible vertex sign configurations. However, 6 of these configurations are ambiguous, leading to non-manifold geometry [S2, unofficial]. Early attempts to fix this included the "Corrected Marching Cubes" lookup table [2518-marching-cubes-lookup-and-configuration.md], which arbitrarily resolved ambiguities. More recent approaches, such as Hermite Dual Marching Cubes [2197-hermite-dual-marching-cubes.md] and Disjoint Patch Decomposition [5004-marching-cubes-with-disjoint-patch-decomposition.md], use dynamic analysis to ensure topological correctness and feature preservation.

## Sources
