# Regularised Marching Tetrahedra


## Overview
Regularised Marching Tetrahedra (RMT) is an iso-surface extraction algorithm that combines the topological consistency of the Marching Tetrahedra (MT) method with vertex clustering to simplify the resulting mesh [S1]. Unlike standard MT, which often produces thin, poorly shaped triangles that degrade interpolated shading, RMT generates surfaces with near-regular triangles and significantly reduced triangle counts (typically 70% fewer than MT) while preserving the original topology [S1]. It builds on [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md) by introducing a pre-triangulation clustering step based on the tetrahedral lattice.

## How it works
**Inputs:**
- A scalar data volume defined on a lattice.
- An iso-value threshold.

**Outputs:**
- A triangle mesh representing the iso-surface.

**Data Structures:**
- **Lattice:** A tessellation of the data volume into tetrahedra. The specific lattice structure is not detailed in the sources, but it must support the definition of tetrahedral edges and vertices [S1].
- **Clustered Vertices:** A set of vertices derived from the original lattice vertices, where nearby vertices are amalgamated.

**Steps:**
1. **Construct Lattice:** Define a suitable lattice to tessellate the data volume into tetrahedra [S1].
2. **Vertex Clustering:** Perform vertex clustering on the sampled data before triangulation. The clustering is based on the original tetrahedral lattice to preserve topology [S1].
   - Vertices are amalgamated based on their proximity within the lattice structure.
   - The position of the amalgamated vertex is not explicitly defined in the sources; standard vertex clustering typically uses the centroid or average of the clustered vertices [G].
   - Note: This clustering is not optimal for minimizing surface area, volume, or curvature errors [S1].
3. **Triangulation:** Triangulate the surface using the clustered vertices. The intersection points of the iso-surface with the tetrahedral edges are calculated using linear interpolation of the sample points at the ends of the edge [S1]. The triangulation follows the standard MT logic but uses the clustered vertices [S1].
4. **Gradient Estimation (Optional):** For shading, estimate the gradient and curvature for a surface vertex using only the data from the vertices of the tetrahedra containing the corresponding edge [S1].

**Formulas:**
- **Intersection Point:** The point of intersection of the iso-surface with an edge is calculated by linear interpolation:
  $$ P = V_1 + \frac{(T - V_1)}{(V_2 - V_1)} (V_2 - V_1) $$
  Where $V_1$ and $V_2$ are the scalar values at the endpoints of the edge, and $T$ is the iso-value. (Standard linear interpolation, implied by [S1]).
- **Gradient Estimation:** The gradient for a vertex is estimated using data from the vertices of the tetrahedra containing the corresponding edge [S1]. The exact formula is not provided in the sources [S1].

**Parameters:**
- **Iso-value:** The threshold for surface extraction.
- **Clustering Threshold:** Not explicitly defined in the sources. The method relies on the lattice structure to determine clustering [S1].

**Edge Cases and Pitfalls:**
- The vertex clustering method is not optimal for minimizing surface area, volume, or surface curvature errors [S1].
- More accurate positioning of amalgamated vertices would require principal curvature calculations, which are not implemented in the basic RMT method [S1].
- The method preserves topology because clustering is based on the tetrahedral lattice, unlike other vertex clustering approaches which may break topology [S1].

## Subtopics

### [Error Analysis via Marching Tetrahedra](1308-error-analysis-via-marching-tetrahedra.md)
Analyzes errors in MT-based methods.

### [Subgrid Pipeline](1640-subgrid-pipeline.md)
A pipeline for subgrid-level processing.

### [McGrids](2289-mcgrids.md)
A grid-based approach for MC.

### [BCC Marching Tetrahedra with Permutation-Based Lookup](3414-bcc-marching-tetrahedra-with-permutation-based-lookup.md)
Uses BCC lattice for MT.

### [Exact Fiber Surface Extraction](3445-exact-fiber-surface-extraction.md)
Focuses on fiber surfaces.

### [Marching Tetrahedra Reconstruction](3516-marching-tetrahedra-reconstruction.md)
Reconstruction techniques for MT.

### [Marching Tetrahedra 3 (DLC-Plane)](3632-marching-tetrahedra-3-dlc-plane.md)
Variant using DLC planes.

### [Raster Algorithm for Fiber Surface Extraction](4960-raster-algorithm-for-fiber-surface-extraction.md)
Raster-based fiber extraction.

## Operating characteristics
- **Time Complexity:** The additional time required for clustering is offset by the reduction in points and triangles stored, resulting in a fast algorithm, comparable to MT alone [S1].
- **Memory Complexity:** Reduced by approximately 70% compared to MT due to fewer vertices and triangles [S1].
- **Performance:** Triangle count is typically reduced by 70% compared to MT and 40% compared to Marching Cubes (MC) [S1].
- **Quality:** Significantly improved triangle aspect ratios, leading to superior interpolated shading (especially Gouraud shading) by eliminating sharp edges and points caused by thin/small triangles [S1].
- **Accuracy:** Volume and surface area error differences compared to MT are much less than the error inherent in the sampling resolution [S1].
- **Strengths:** Topological consistency, reduced triangle count, improved shading quality, fast execution.
- **Limitations:** Clustering is not optimal for minimizing geometric errors (area, volume, curvature); requires careful handling of lattice structure for clustering.

## Background
Regularised Marching Tetrahedra was introduced to address the issues of poor triangle aspect ratios and high triangle counts in standard Marching Tetrahedra and Marching Cubes, particularly for smooth shaded displays [S1]. It combines MT with vertex clustering, a technique previously used for mesh simplification but often lacking topological consistency [S1]. The method is suitable for implicit or discrete data, including medical data and object cross-sections [S1].

## Sources
[S1] Regularised marching tetrahedra: improved iso-surface extraction, https://doi.org/10.1016/s0097-8493(99)00076-x, fetched 2026-10-27
