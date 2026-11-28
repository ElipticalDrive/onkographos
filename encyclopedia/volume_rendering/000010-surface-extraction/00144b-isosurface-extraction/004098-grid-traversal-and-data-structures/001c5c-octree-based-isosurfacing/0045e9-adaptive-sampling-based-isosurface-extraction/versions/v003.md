# Adaptive Sampling-based Isosurface Extraction

Also called: Octree Dual Contouring (OctDC), Octree Marching Cubes (OctMC)

## Overview
Adaptive sampling-based isosurface extraction constructs a mesh by sampling the implicit scalar field at discrete locations (typically octree leaves) rather than computing analytic intersections within grid cells. This approach sacrifices geometric precision for computational efficiency and ease of implementation on adaptive structures. It differs from its siblings by relying on discrete point clouds or simplified connectivity derived from samples, whereas methods like [Wilhelms and Van Gelder Octree](2107-wilhelms-and-van-gelder-octree.md) or [Standard Interval Tree for Isosurface Extraction](1230-standard-interval-tree-for-isosurface-extraction.md) often preserve topological guarantees or analytic intersection logic within the spatial hierarchy.

## Shared principles
The family shares the mechanism of using an octree to adaptively resolve the scalar field $V$. The isosurface is defined by the set of sample points (usually octree leaves) where the field value crosses the isovalue $v$. Unlike analytic methods that compute exact intersection coordinates on cell edges, these methods approximate the surface geometry using the sampled locations or simplified primitives associated with the leaves. The primary challenge is maintaining topological consistency (avoiding cracks) and geometric accuracy when the sampling density varies across the surface.

## Subtopics

### [Isosurface Approximation via Grid Cell Center Sampling](15426-isosurface-approximation-via-grid-cell-center-sampling.md)
Approximates the surface by placing a single point at the center of every grid cell intersecting the isosurface. It sacrifices precision for speed, bounding the distance error by half the cell diagonal ($\frac{\sqrt{3}}{2}$ for unit cells). Choose this when extreme efficiency is required and exact geometry is secondary, such as in similarity map computation.
- [Octree Marching Cubes (OctMC)]: Uses the Marching Cubes topology logic adapted to octree leaves. It generates more vertices than analytic methods and can suffer from topological inconsistencies if the octree is not strictly adapted to the surface.
- [Octree Dual Contouring (OctDC)]: Uses dual contouring logic on octree leaves to preserve sharp features. It typically produces higher vertex counts than OctMC but requires solving linear systems for vertex positions.
- [McGrids]: A grid-based adaptive method that balances between full analytic extraction and coarse sampling.

## Background
The term "Adaptive Sampling-based Isosurface Extraction" is used in recent literature to categorize methods like OctMC, OctDC, and McGrids that rely on discrete sampling within an octree hierarchy [S1]. These methods are often contrasted with "Analytic Isosurface Extraction" (e.g., TetraSDF), which computes exact intersections [S1].

Historical context: Early octree isosurfacing methods like [Wilhelms and Van Gelder Octree](2107-wilhelms-and-van-gelder-octree.md) established the foundation for adaptive spatial decomposition. Later, [Ashida and Badla Octree Contouring](2117-ashida-and-badla-octree-contouring.md) and [Unconstrained Isosurface Extraction on Arbitrary Octrees](2413-unconstrained-isosurface-extraction-on-arbitrary-octrees.md) addressed topological issues. The specific comparison in [S1] highlights that while these adaptive sampling methods are standard, they can be computationally expensive and produce excessive vertex counts for high-resolution implicit fields.

Performance data from [S1] on the Stanford Bunny dataset at depth 12 shows that OctMC takes 124.2s and produces 6.76M vertices, while OctDC takes 139.3s and produces 13.49M vertices. In contrast, analytic methods (TetraSDF) achieved 4.56s and 0.28M vertices. The study notes that at depth 12, octree-based extraction became impractical for the Bunny model, producing 71M leaves, indicating that octree adaptivity alone does not prevent near-surface explosion on curved shapes [S1].

## Sources
[S1] TetraSDF: Analytic Isosurface Extraction with Multi-resolution Tetrahedral Grid, https://arxiv.org/abs/2511.16273v4, fetched 2026-10-26
