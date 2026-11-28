# Hierarchical Data Structure Traversal

Also called: Ray-Casting with Hierarchical Data Structure (Ma)

## Overview
Hierarchical Data Structure Traversal is a parallel ray-casting method for rendering unstructured volume data, specifically designed to handle complex cell connectivity using a hierarchical data structure (HDS) [S1]. It solves the problem of efficiently traversing unstructured grids by pre-computing face-node and cell-face relationships, allowing rays to jump between cells based on exterior face intersections rather than dense voxel stepping [S1]. Unlike sibling methods that rely on regular grid mips or spatial partitioning trees (e.g., [KD-Tree Partitioned Volume Rendering](1966-kd-tree-partitioned-volume-rendering.md), [Octree-based traversal](5348-octree-based-traversal.md)), this approach explicitly manages the topological connectivity of arbitrary unstructured cells to determine ray entry and exit points [S1].

## How it works
**Inputs:** Unstructured volume dataset (cells, nodes, faces); camera parameters; hierarchical data structure (HDS) containing face-node, cell-face, and cell-node relationships [S1].
**Outputs:** Composite image pixels [S1].

**Data Structures:**
The HDS stores topological relationships:
1.  `face_node`: Maps each face to its constituent nodes.
2.  `cell_face`: Maps each cell to its constituent faces.
3.  `cell_node`: Maps each cell to its constituent nodes.
4.  `exterior_faces`: A list of faces that are not shared between two cells (boundary faces) [S1].

**Steps:**
1.  **Connectivity Pre-step:** Each processor computes the cell connectivity information and identifies exterior faces [S1].
2.  **View Frustum Culling:** Cells completely outside the camera view are excluded from processing [S1].
3.  **Ray Casting Application:** Each processor applies the ray-casting algorithm to its assigned subset of data [S1].
4.  **Ray Traversal:**
    a.  A ray enters the volume through an exterior face [S1].
    b.  The algorithm uses the HDS connectivity information to determine the next cell intersected by the ray [S1].
    c.  The ray continues traversing cells until it intersects a second exterior face, at which point it exits the volume [S1].
5.  **Parallel Composition:** Image pixels are assigned to different processors. The HDS is used to exchange ray data between processors if necessary [S1].
6.  **Compositing:** Rays are composited (blended) to produce the final pixel color [S1].

**Formulas:**
The value of each sample is computed by interpolating cell vertices. The corresponding scalar value is used as a 1D texture coordinate to obtain the color [S1].
$$ C = \text{Interpolate}(V_{nodes}) $$
$$ \text{Color} = \text{TextureLookup}(C) $$
Slices are rendered in back-to-front order based on distance from the viewing direction [S1].

**Parameters:**
*   **Processor Count:** Determined by the parallel implementation (many-core sampler) [S1].
*   **Slicing Strategy:** Three slices per cell (perpendicular to X, Y, Z axes) are generated; the camera view determines which slice is used [S1].

**Edge Cases:**
*   **Exterior Face Identification:** Correctly identifying faces that are not shared between cells is critical for determining ray entry/exit [S1].
*   **Parallel Exchange:** The HDS must support efficient ray exchange between processors assigned to different image pixels [S1].

## Subtopics

### [Hypertree Cursor](18831-hypertree-cursor.md)
A structure pointing to a vertex of a hypertree within a hypertree grid, capable of accessing vertex attributes and moving to connected vertices. It solves the problem of navigating tree-based Adaptive Mesh Refinement (AMR) grids by providing a flexible traversal mechanism that does not impose a predetermined scheme, unlike iterators. It sets itself apart from simple iterators by supporting arbitrary displacement operations (such as moving to parents or roots) rather than just sequential iteration.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Depends on the storage of the HDS (face-node, cell-face, cell-node relationships) [S1].
*   **Performance:** Designed for scalability on many-core architectures using a hybrid scheme [S1].
*   **Strengths:** Efficient handling of unstructured data via explicit connectivity; parallelizable via processor assignment [S1].
*   **Limitations:** Requires pre-computation of connectivity; relies on HDS for inter-processor communication [S1].
*   **When to use:** Rendering unstructured volume data where regular grid traversal is inefficient or impossible [S1].
*   **When not to use:** Regular structured grids where simpler DDA or grid-based traversal is sufficient [S1].

## Background
Originated in the work of Max et al. [21] and adapted by the authors of [S1] using a many-core sampler (EAVL) implemented by Larsen et al. [22] [S1]. The method was developed to address the challenges of rendering unstructured volume data in a parallel environment [S1].

## Sources
[S1] A Scalable Hybrid Scheme for Ray-Casting of Unstructured Volume Data, https://doi.org/10.1109/tvcg.2018.2833113, fetched 2026-10-24
