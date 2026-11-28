# Feature Edges Representation

Also called: Feature Edges

## Overview
Feature Edges Representation renders only the prominent geometric features of a surface mesh, such as sharp creases or boundary edges, rather than the entire surface topology. It serves to highlight structural discontinuities in volumetric data without the visual clutter of a full wireframe or solid surface. It is distinguished from its sibling [Wireframe Representation](772-line-illustration-detection.md) by being a strict subset that filters out smooth, continuous cell boundaries, retaining only those with significant geometric significance.

## How it works
Feature Edges relies on the extraction of a surface mesh from the underlying dataset, followed by a filtering process to identify "prominent" edges. The method operates on the connectivity of the mesh cells (typically triangles or quadrilaterals).

**Inputs and Outputs:**
*   **Input:** A surface mesh (extracted from the volume) and a scalar data array (optional, for coloring).
*   **Output:** A set of line segments representing the prominent edges.

**Data Structures:**
The core data structure is the mesh connectivity graph. For a mesh composed of triangular cells, each edge is defined by two vertices. The algorithm tracks the adjacency of cells sharing an edge.

**Algorithm Steps:**
1.  **Mesh Extraction:** Extract the surface mesh from the volumetric dataset. This involves identifying the boundary between cells with data and cells without, or using an isosurface extraction algorithm (e.g., Marching Cubes) if the input is raw scalar data [S1].
2.  **Edge Identification:** Iterate through all edges in the mesh. An edge is defined by the pair of vertices it connects.
3.  **Prominence Filtering:** For each edge, determine if it is "prominent" based on one of two criteria [S1]:
    *   **Sharp Angle:** Calculate the dihedral angle between the two adjacent cells sharing the edge. If the angle exceeds a predefined threshold (indicating a sharp crease), the edge is selected.
    *   **Boundary Edge:** Check if the edge is shared by only one cell (i.e., it lies on the boundary of the mesh). If so, the edge is selected.
4.  **Rendering:** Render the selected edges as line primitives.
5.  **Coloring:** Apply a single solid color or a scalar data array (pseudocoloring) to the edges [S1].

**Formulas:**
Let $E$ be the set of edges in the mesh. For an edge $e \in E$, let $C_1$ and $C_2$ be the cells adjacent to $e$.
*   If $C_1$ and $C_2$ exist, calculate the dihedral angle $\theta$ between their normal vectors $\mathbf{n}_1$ and $\mathbf{n}_2$:
    $$ \theta = \arccos(\mathbf{n}_1 \cdot \mathbf{n}_2) $$
    If $\theta > \theta_{threshold}$, $e$ is a feature edge.
*   If only one cell $C_1$ is adjacent to $e$, $e$ is a boundary feature edge.

**Parameters:**
*   **Angle Threshold:** The minimum dihedral angle to consider an edge "sharp." Not given in the sources.
*   **Color Mode:** Either solid color or scalar-based [S1].

**Edge Cases and Pitfalls:**
*   **Degenerate Cells:** Cells with zero area or coincident vertices may cause undefined normals. [G] Ensure normals are computed robustly or degenerate cells are removed during mesh extraction.
*   **Mesh Quality:** The accuracy of the angle calculation depends on the quality of the extracted surface mesh. Noisy meshes may produce false feature edges.

## Subtopics

### Wireframe Representation
Wireframe Representation renders the entire surface mesh as a collection of cell boundaries. It is the superset of Feature Edges. While Feature Edges filters for prominence (sharp angles or boundaries), Wireframe displays all edges regardless of geometric significance. Choose Wireframe when the complete topological structure is needed; choose Feature Edges when only structural highlights are desired to reduce visual clutter [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of edges in the surface mesh, $O(E)$, where $E$ is the number of edges. The dominant cost is mesh extraction and edge adjacency checks.
*   **Memory Complexity:** Linear with respect to the number of edges, $O(E)$, to store the selected edge indices.
*   **Strengths:** Reduces visual clutter compared to full wireframe or solid surface rendering; highlights key structural features like creases and boundaries.
*   **Limitations:** Does not convey the full surface shape or smoothness; relies on accurate mesh extraction; may miss subtle features if the angle threshold is too high.
*   **When to use:** When the user needs to understand the underlying geometric structure or discontinuities of a volume without the occlusion of a solid surface.
*   **When not to use:** When the smooth variation of the data is more important than the structural boundaries, or when the mesh extraction is unreliable.

## Background
Feature Edges Representation is a visualization technique commonly implemented in scientific visualization tools such as ParaView. It is part of the broader category of wireframe-based rendering methods. The concept is documented in the ParaView user guide, which describes it as a subset of wireframe rendering that focuses on prominent edges [S1, S2].

## Sources
[S1] 4. Displaying data — ParaView Documentation 5.8.1 documentation, https://docs.paraview.org/en/v5.9.1/UsersGuide/displayingData.html, fetched 2026-10-29
[S2] 5. Displaying data — ParaView Documentation 6.2.0 documentation, https://docs.paraview.org/en/latest/UsersGuide/displayingData.html, fetched 2026-10-29
