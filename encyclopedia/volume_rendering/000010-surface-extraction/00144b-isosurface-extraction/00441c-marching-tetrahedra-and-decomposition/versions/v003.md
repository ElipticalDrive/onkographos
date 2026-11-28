# Marching Tetrahedra and Decomposition

Also called: Tetrahedral Marching, MC-Tet

## Overview
Marching Tetrahedra and Decomposition resolves the topological ambiguities inherent in standard Marching Cubes by decomposing each cubic voxel into a fixed set of tetrahedra, as implemented in [Marching Cubes Tetrahedron Decomposition](9029-marching-cubes-tetrahedron-decomposition.md). Unlike standard Marching Cubes, which requires complex disambiguation logic for ambiguous cases, this approach ensures a unique intersection topology for each of the 16 possible vertex configurations within a tetrahedron [G]. It sets itself apart from [Subgrid Marching Tetrahedra](1737-subgrid-marching-tetrahedra.md) by operating on the primary voxel grid without sub-voxel refinement, and from [Error Analysis via Marching Tetrahedra](1308-error-analysis-via-marching-tetrahedra.md) by focusing on the geometric decomposition strategy itself rather than error metrics or topological fusion bounds.

## Shared principles
All methods in this family operate on a discretized domain, typically a regular grid (Cartesian, octree, or tetrahedral) where each cell contains scalar values or occupancy states at its vertices or centers [G]. For detailed traversal, surface determination, and mesh coherence mechanisms, see [Surface extraction](29-surface-extraction.md).

## Subtopics

### [Error Analysis via Marching Tetrahedra](1308-error-analysis-via-marching-tetrahedra.md)
Evaluates the geometric and topological approximation error when fusing two child tetrahedra into a single parent tetrahedron within a multiresolution hierarchy [1308-error-analysis-via-marching-tetrahedra.md]. It determines whether a coarser representation preserves the isosurface geometry within a user-defined tolerance [1308-error-analysis-via-marching-tetrahedra.md]. Choose this method when verifying that multiresolution fusion does not alter surface connectivity or exceed a physical distance threshold [1308-error-analysis-via-marching-tetrahedra.md].

### [Subgrid Marching Tetrahedra](1737-subgrid-marching-tetrahedra.md)
Recovers a manifold, intersection-free triangle mesh from the points where edges of a tetrahedral grid pierce a continuous surface [1737-subgrid-marching-tetrahedra.md]. It solves the limitation of classic marching methods by allowing arbitrarily many surface patches within a single cell to capture fine features and thin sheets [1737-subgrid-marching-tetrahedra.md]. Choose this method when the input geometry contains open curves, thin sheets, or multiple intersections per edge that standard lookup tables cannot capture [1737-subgrid-marching-tetrahedra.md].

### [Marching Cubes Tetrahedron Decomposition](9029-marching-cubes-tetrahedron-decomposition.md)
Resolves the topological ambiguities inherent in standard Marching Cubes by decomposing each cubic voxel into a fixed set of tetrahedra and applying a 16-case lookup table for edge interpolation [9029-marching-cubes-tetrahedron-decomposition.md]. It differs from [Subgrid Marching Tetrahedra](1737-subgrid-marching-tetrahedra.md) by operating on the primary voxel grid without sub-voxel refinement, and from [Error Analysis via Marching Tetrahedra](1308-error-analysis-via-marching-tetrahedra.md) by focusing on the geometric decomposition strategy itself rather than error metrics [9029-marching-cubes-tetrahedron-decomposition.md]. Choose this method when standard Marching Cubes ambiguity resolution is insufficient and a deterministic, lookup-table-driven approach on a regular grid is preferred [9029-marching-cubes-tetrahedron-decomposition.md].

### [TetraSDF Marching Tetrahedra Extraction](13238-tetrasdf-marching-tetrahedra-extraction.md)
Generates a triangular mesh from a trained Signed Distance Function (SDF) neural network by evaluating the network on a multi-resolution tetrahedral grid [13238-tetrasdf-marching-tetrahedra-extraction.md]. It differs from standard grid-based marching tetrahedra by using an adaptive, multi-resolution domain decomposition and an analytic intersection approach to handle the continuous nature of the neural network output [13238-tetrasdf-marching-tetrahedra-extraction.md]. Choose this method when extracting surfaces directly from learned implicit representations without discrete voxel sampling [13238-tetrasdf-marching-tetrahedra-extraction.md].

## Background
The specific origin (authors, date, paper) is Not given in the sources. The method resolves the topological ambiguities present in the original Marching Cubes algorithm, which could produce non-manifold geometries in specific ambiguous cases [9029-marching-cubes-tetrahedron-decomposition.md]. By decomposing the cube into tetrahedra, the problem is reduced to the 16 unambiguous cases of a tetrahedron [9029-marching-cubes-tetrahedron-decomposition.md]. It is widely used in medical imaging and scientific visualization where topological correctness is critical [9029-marching-cubes-tetrahedron-decomposition.md].

## Sources
