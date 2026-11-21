# Topological Landscape


## Overview
Topological landscape is a visualization method that maps the topological structure of a scalar field's contour tree onto a 3D terrain model to enhance the perceptual clarity of volumetric features [S1, unofficial]. It solves the problem of interpreting complex topological changes (merges and splits) in high-dimensional data by leveraging human intuition for reading terrain structures [S1, unofficial]. It sets itself apart from direct volume rendering by abstracting the scalar field into a structural graph representation, and from other contour-based methods like [Outline and Contour Generation](4137-outline-and-contour-generation.md) by explicitly encoding topological events as geometric elevations rather than screen-space edges.

## How it works
**Inputs and Outputs:**
*   **Input:** A contour tree $T$ derived from a scalar field $f(x,y,z)$. The tree consists of nodes representing critical points (minima, maxima, saddles) and edges representing intervals between them [S1, unofficial].
*   **Output:** A 3D terrain mesh where the x and y coordinates represent the topological position within the tree, and the z coordinate represents the scalar value (isovalue) [S1, unofficial].

**Data Structures:**
The core data structure is the contour tree, typically represented as a graph where nodes $v_i$ have attributes:
*   `scalar_value`: The scalar value at the critical point.
*   `type`: Minimum, Maximum, or Saddle.
*   `children`: List of connected nodes with higher scalar values.
*   `parents`: List of connected nodes with lower scalar values.

**Steps:**
1.  **Construct Contour Tree:** Compute the contour tree for the input scalar field. This identifies all critical points and the connectivity of level sets [S1, unofficial].
2.  **Layout Mapping:** Map the tree structure to a 2D plane. This step uses established 2D visualization techniques to lay out the tree nodes, minimizing edge crossings and preserving hierarchical relationships [S1, unofficial]. Let the 2D position of node $v$ be $(x_v, y_v)$.
3.  **Terrain Generation:** Extrude the 2D layout into 3D. For each node $v$, the 3D position is $(x_v, y_v, z_v)$, where $z_v = f(v)$ is the scalar value at that critical point [S1, unofficial].
4.  **Mesh Construction:** Connect the 3D points to form a surface mesh. Edges in the contour tree correspond to ridges or valleys in the terrain.
    *   **Valleys:** Represent events where a contour splits into two or more parts (e.g., a saddle point leading to multiple maxima).
    *   **Peaks:** Represent events where two or more contours merge (e.g., multiple minima leading to a single maximum) [S1, unofficial].
5.  **Rendering:** Render the resulting terrain using standard surface rendering techniques. Peaks in the landscape correspond to distinct topological features in the direct volume rendering [S1, unofficial].

**Formulas:**
*   Terrain Height: $z = f(v)$ for each critical point $v$.
*   Topological Event Identification:
    *   Split: Node with one parent and multiple children (in the direction of increasing scalar value).
    *   Merge: Node with multiple parents and one child (in the direction of increasing scalar value) [S1, unofficial].

**Parameters:**
*   Layout Algorithm: Various 2D layout techniques can be applied; specific parameters depend on the chosen layout method [S1, unofficial].
*   Scalar Range: The full range of the scalar field $f$ determines the vertical scale of the landscape [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **High-Dimensional Data:** The method can be applied to contour trees of any number of dimensions, but layout complexity increases significantly [S1, unofficial].
*   **Ambiguity in Layout:** The choice of 2D layout technique affects the interpretability of the terrain; poor layouts may obscure topological relationships [S1, unofficial].

## Subtopics

### [Contour Tree](9312-contour-tree.md)
The underlying topological data structure used to build the landscape. It captures changes in the topology of level sets as the isovalue sweeps across the scalar range. It is chosen when a precise, one-to-one correspondence between tree edges and contours is required for a simply connected domain.

## Operating characteristics
*   **Time Complexity:** Dominated by the construction of the contour tree, which is typically $O(N \log N)$ or $O(N)$ depending on the algorithm, where $N$ is the number of voxels or cells [S1, unofficial]. The layout step depends on the specific 2D visualization technique used.
*   **Memory Complexity:** Proportional to the size of the contour tree, which is generally much smaller than the original volume data, making it memory efficient for large datasets [S1, unofficial].
*   **Strengths:** Provides an intuitive, compact representation of complex topological structures; effective for exploring high-dimensional data; leverages human terrain perception [S1, unofficial].
*   **Limitations:** Requires pre-computation of the contour tree; the 2D layout step can be subjective and may not always preserve all topological nuances; not suitable for real-time interactive exploration without pre-processing [S1, unofficial].
*   **When to Use:** When the primary goal is to understand the global topological structure of a scalar field, especially in high-dimensional or complex datasets where direct volume rendering is ambiguous.
*   **When Not to Use:** When detailed local geometric features are more important than topological structure, or when real-time performance is critical without pre-computation.

## Background
Topological landscape was proposed by Weber et al. [45] as an alternative approach to viewing contour information [S1, unofficial]. It was further expanded using different methods for laying out the data, primarily from established 2D visualization techniques [S1, unofficial]. The method is part of the broader field of topological visualization techniques for volume multifield data [S1, unofficial].

## Sources
[S1] Chapter Topological Visualisation Techniques for Volume Multifield Data, https://archive.org/details/oapen-20.500.12657-49310, fetched 2026-10-27 (unofficial)
