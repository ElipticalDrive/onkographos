# Adaptive Sampling-based Isosurface Extraction

Also called: Octree Dual Contouring (OctDC), Octree Marching Cubes (OctMC)

## Overview
Adaptive sampling-based isosurface extraction constructs a mesh by sampling the implicit scalar field at discrete locations rather than computing analytic intersections [S1]. It differs from its siblings by relying on discrete point clouds or simplified connectivity derived from samples, whereas methods like [Wilhelms and Van Gelder Octree](2107-wilhelms-and-van-gelder-octree.md) or [Standard Interval Tree for Isosurface Extraction](1230-standard-interval-tree-for-isosurface-extraction.md) often preserve topological guarantees or analytic intersection logic within the spatial hierarchy [G].

## Shared principles
The family relies on adaptive sampling of the implicit field to construct a mesh, as opposed to analytic extraction [S1]. It uses an octree to adaptively resolve the scalar field $V$, defining the isosurface by sample points (typically octree leaves) where the field value crosses the isovalue $v$ [G]. Unlike analytic methods that compute exact intersection coordinates on cell edges, these methods approximate the surface geometry using sampled locations or simplified primitives associated with the leaves [G]. The primary challenge is maintaining topological consistency (avoiding cracks) and geometric accuracy when sampling density varies across the surface [G].

## Subtopics

### [Isosurface Approximation via Grid Cell Center Sampling](15426-isosurface-approximation-via-grid-cell-center-sampling.md)
Approximates an isosurface $S$ for a given isovalue $v$ by generating exactly one sample point per 3D grid cell that intersects the surface, replacing exact analytic extraction to accelerate distance field computation. It differs from its siblings in the [Adaptive Sampling-based Isosurface Extraction](11618-adaptive-sampling-based-isosurface-extraction.md) family by using a fixed grid cell topology and a simple center-point heuristic rather than adaptive octree leaf refinement or complex primitive construction, prioritizing computational speed and GPU parallelism over geometric precision.

## Background
The term "Adaptive Sampling-based Isosurface Extraction" categorizes methods like OctMC, OctDC, and McGrids that rely on discrete sampling within an octree hierarchy [S1]. These methods are often contrasted with Analytic Marching (AM), which computes exact intersections [S1]. The comparison in [S1] highlights that these adaptive sampling methods can be computationally expensive and produce excessive vertex counts for high-resolution implicit fields. Performance data on the Stanford Bunny dataset at depth 12 shows that OctMC takes 124.2s and produces 6.76M vertices, while OctDC takes 139.3s and produces 13.49M vertices [S1]. In contrast, the analytic method (Ours) achieved 4.56s and 0.28M vertices [S1]. The study notes that at depth 12, octree-based extraction became impractical for the Bunny model, producing 71M leaves, indicating that octree adaptivity alone does not prevent near-surface explosion on curved shapes [S1].

## Sources
[S1] TetraSDF: Analytic Isosurface Extraction with Multi-resolution Tetrahedral Grid, https://arxiv.org/abs/2511.16273v4, fetched 2026-10-26
