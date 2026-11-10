# Topological Correctness Guarantees

Also called: Topologically Correct Marching Cubes, Ambiguity Resolution

## Overview
Topological Correctness Guarantees is a family of methods designed to resolve the topological ambiguities inherent in the standard Marching Cubes algorithm [774-marching-cubes.md] [G]. Standard Marching Cubes relies on a simplified lookup table that fails to distinguish between multiple valid topologies for certain vertex configurations, potentially producing surfaces that are not homeomorphic to the underlying trilinear interpolant (e.g., representing a tunnel as two disjoint sheets) [1368-topologically-correct-marching-cubes.md]. This family ensures the extracted mesh is a 2-manifold and watertight by using higher-order interpolation tests, combinatorial connectedness constraints, or extended triangulation rules to select the topology that matches the continuous scalar field [1368-topologically-correct-marching-cubes.md][1428-topologically-defined-isosurfaces.md].

## Shared principles
All methods in this family operate on the voxel grid cells processed by Marching Cubes [G]. They share the following mechanism:
1. **Ambiguity Detection:** Identify voxel configurations where the binary signs of the 8 vertices are insufficient to determine a unique topology for the isosurface intersection [G]. In standard Marching Cubes, these are the "ambiguous cases" (e.g., Case 6 and Case 9 in 2D, and specific complex cases in 3D like Case 13) [790-asymptotic-decider.md][1618-extended-trilinear-triangulation.md].
2. **Topology Selection:** Apply a decision criterion to resolve the ambiguity [G]. This criterion must be consistent with the underlying continuous function defined by the interpolation model (linear, bilinear, trilinear, or Hermite) [790-asymptotic-decider.md][961-marching-cubes-with-hermite-data-and-full-case-table.md].
3. **Manifold Guarantee:** The selected topology must result in a mesh where every edge is shared by exactly two triangles (2-manifold) and the surface is closed (watertight), provided the isosurface does not intersect the domain boundaries [1368-topologically-correct-marching-cubes.md].

## Subtopics

### [Asymptotic Decider](790-asymptotic-decider.md)
Uses bilinear interpolation in 2D to compute a decider value $V$ that distinguishes between the two ambiguous hyperbolic topologies. Chosen for 2D applications or as a component in 3D face ambiguity resolution.

### [Marching Cubes with Hermite Data and Full Case Table](961-marching-cubes-with-hermite-data-and-full-case-table.md)
Extends MC with second-order Hermite interpolation and a full 256-case table to resolve 3D ambiguities using gradient information. Chosen when higher geometric accuracy and explicit handling of all cases are required.

### [Topologically Correct Marching Cubes](1368-topologically-correct-marching-cubes.md)
Guarantees the mesh is homeomorphic to the trilinear interpolant by inserting additional vertices in ambiguous cells to resolve tunnels. Chosen for general 3D volume rendering where topological fidelity to the trilinear field is critical.

### [Topologically Defined Isosurfaces](1428-topologically-defined-isosurfaces.md)
Extracts surfaces from binary volumes using connectedness couples $(\eta, \lambda)$ and local graph construction to ensure consistency with the inverse image. Chosen for binary volumetric data requiring strict topological correctness without interpolation.

### [Topologically Defined Isosurface via Connectedness Couples](1430-topologically-defined-isosurface-via-connectedness-couples.md)
Similar to the above but uses a precomputed local configuration table based on valid connectedness couples to generate the triangulation. Chosen for efficient extraction from binary volumes using combinatorial manifold definitions.

### [Extended Trilinear Triangulation](1618-extended-trilinear-triangulation.md)
Extends MC33 with ternary vertex labeling ('+', '-', '=') and convex hull triangulation of vertex/edge groups to resolve trilinear ambiguities and avoid degenerate triangles. Chosen for robust handling of degenerate cases and complex topologies in trilinear fields.

### [Topology/Normal Consistency Loss](4351-topology-normal-consistency-loss.md)
A learned loss function for neural networks that enforces topological and normal consistency in predicted marching cubes outputs. Chosen for differentiable rendering and learned surface extraction.

## Background
The standard Marching Cubes algorithm by Lorensen and Cline (1987) uses a 12-case lookup table, ignoring the topological ambiguities present in the remaining 256-12=244 cases [774-marching-cubes.md] [G]. This led to topological errors, such as the "tunnel" problem where a single tunnel is rendered as two separate sheets [G]. Early work by Williams (1990) and others identified these ambiguities [G]. The Asymptotic Decider was introduced to resolve 2D ambiguities using bilinear interpolation [790-asymptotic-decider.md]. In 3D, various approaches emerged: [1368-topologically-correct-marching-cubes.md] proposed inserting vertices to match the trilinear interpolant; [1618-extended-trilinear-triangulation.md] (MC33) used a 33-case table based on trilinear analysis; and [1428-topologically-defined-isosurfaces.md] / [1430-topologically-defined-isosurface-via-connectedness-couples.md] focused on binary volumes using digital topology and connectedness couples. [961-marching-cubes-with-hermite-data-and-full-case-table.md] explored higher-order interpolation for geometric accuracy.

## Sources
[S1] Asymptotic Decider, https://example.com/asymptotic-decider, 2023-10-01
[S2] Asymptotic Decider Unofficial, https://example.com/asymptotic-decider-unofficial, 2023-10-01
