# Vertex Clustering for Adaptive DC

Also called: Dual Contouring with Topology-Preserving Simplification (Zhang et al.), Zhang et al. Vertex Clustering for Adaptive DC, Zhang's adaptive DC variant

## Overview
Vertex Clustering for Adaptive DC is a modified dual contouring algorithm that employs topology-preserving simplification to maintain disconnected surface components within octree cells during isosurface extraction [S1]. Unlike standard adaptive dual contouring which may merge distinct components or fail to preserve topology under simplification, this method uses an enhanced cell representation to ensure that multiple disconnected contour patches are handled separately, improving adaptivity for complex geometries [S1]. It differs from later manifold-preserving variants by allowing multiple components per cell but lacking the strict 2-manifold guarantees of those successors [S2, unofficial].

## How it works
**Inputs:** A scalar field defined on a grid; an octree structure representing the adaptive subdivision of the domain.
**Outputs:** A triangle mesh approximating the isosurface, with vertices and connectivity information.

**Data Structures:**
*   **Enhanced Cell:** A data structure representing a single octree leaf or internal node. It contains:
    *   A list of surface components.
    *   Each surface component is represented by a vertex (position) and its connectivity information (indices to adjacent vertices/edges).
*   **Vertex Octree:** The hierarchical structure built by clustering vertices from the finest resolution, preserving the topology of disconnected components.

**Steps:**
1.  **Enhanced Cell Representation:** Identify isosurface components within each cell. Represent each disconnected component as a vertex with associated connectivity data in an enhanced cell structure [S1].
2.  **Topology-Preserving Vertex Clustering:** Apply a vertex clustering algorithm to build a vertex octree. This step simplifies the representation by clustering vertices from finer levels into coarser levels while preserving the topological separation of disconnected components [S1].
3.  **Connectivity-Guided Simplification:** Use a connectivity-guided vertex clustering algorithm to simplify the isosurface components. This ensures that the simplification process respects the connectivity of the surface patches [S1].
4.  **Enhanced Dual Contouring Extraction:** Employ an enhanced dual contouring algorithm to extract error-bounded multi-resolution isosurfaces from the vertex octree. This step extracts the final mesh while preserving the finest resolution isosurface topology [S1].

**Parameters:**
*   Error bound: Determines the level of detail and simplification. Not given in the sources.
*   Maximum intersections per edge: Restricted to 2. This is a hard constraint of the method [S2, unofficial].

**Pitfalls:**
*   **Non-Manifold Geometry:** The method can generate non-manifold edges and vertices in adaptive settings [S2, unofficial].
*   **Intersection Cap:** The method restricts the maximum number of intersections between the contour and each cell edge to 2. This limits the complexity of contours that can be handled [S2, unofficial].
*   **Implementation Complexity:** Requires non-trivial coding-vector operations for the connectivity-guided clustering [S2, unofficial].

## Subtopics

### [Manifold Dual Contouring](2138-manifold-dual-contouring.md)
A variant that guarantees manifold surfaces even under adaptive simplification using a different topology constraint. Choose when manifoldness is critical.

### [Adaptive Contouring via Octree Vertex Clustering](2142-adaptive-contouring-via-octree-vertex-clustering.md)
A simpler method with no limit on intersections per edge. Choose for simpler implementation and less restrictive simplification.

### [Adaptive Dual Contouring](3664-adaptive-dual-contouring.md)
General adaptive dual contouring without specific topology-preserving simplification. Choose for basic adaptive extraction.

### [Kdtree-based Iso-surface Extraction](3816-kdtree-based-iso-surface-extraction.md)
Uses a k-d tree instead of an octree. Choose for different spatial partitioning needs.

### [Mesh Extraction from K-D Tree](4607-mesh-extraction-from-k-d-tree.md)
Similar to above, focusing on mesh extraction from k-d tree structures.

### [Isomesh Octree Build and Clustering](4984-isomesh-octree-build-and-clustering.md)
Focuses on octree build and clustering for isomeshes.

### [Intersection-Free Dual Contouring](5313-intersection-free-dual-contouring.md)
Ensures intersection-free contours. Choose for robustness against self-intersections.

### [Octree Construction for Dual Contouring](5913-octree-construction-for-dual-contouring.md)
Focuses on the octree construction phase.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Results in much better adaptivity than standard Dual Contouring [S1].
*   **Strengths:** Preserves disconnected surface components, improving adaptivity for complex geometries [S1].
*   **Limitations:** Can generate non-manifold geometry; restricted to max 2 intersections per edge; requires complex coding-vector operations [S2, unofficial].
*   **When to use:** When preserving disconnected components is more important than strict manifoldness, and the geometry does not exceed 2 intersections per edge.
*   **When not to use:** When manifoldness is required, or when the contour has more than 2 intersections per edge.

## Background
Proposed by Zhang et al. as a modification to the original Dual Contouring algorithm by Ju et al. [S1]. The original DC algorithm used an octree for adaptive representation, but Zhang et al. aimed to improve performance by preserving disconnected surface components during simplification [S1]. This method was an early attempt to handle multiple components per cell in adaptive dual contouring, preceding later manifold-preserving variants [S2, unofficial].

## Sources
[S1] Evaluation of Manifold Dual Contouring Algorithms Based on K-d Tree and Octree Data Structures, https://doi.org/10.31449/inf.v48i3.5420, fetched 2026-10-26
[S2] PDF Manifold Dual Contouring, https://people.engr.tamu.edu/schaefer/research/dualsimp_tvcg.pdf, fetched 2026-10-27 (unofficial)
