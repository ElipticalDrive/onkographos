# Topological Landscape


## Overview
Topological Landscape is a visualization technique that maps the topology of a scalar field onto a 3D terrain model to enhance the interpretability of contour relationships [S1, unofficial]. It builds on [Outline and Contour Generation](4137-outline-and-contour-generation.md) by converting abstract graph structures into an intuitive spatial representation where valleys indicate contour splits and peaks indicate contour merges [S1, unofficial]. Unlike [Object Contour NPR](787-object-contour-npr.md) which renders lines directly on the volume, this method provides a meta-visualization of the data's structural hierarchy.

## How it works
**Inputs and Outputs:**
*   **Input:** A contour tree representing the scalar field's topology [S1, unofficial].
*   **Output:** A 3D terrain model (height field) where the geometry encodes topological events [S1, unofficial].

**Data Structures:**
The core data structure is the contour tree, a graph where nodes represent critical points (minima, maxima, saddles) and edges represent intervals of constant topology [S1, unofficial]. The landscape maps these nodes to 3D coordinates $(x, y, z)$.

**Steps:**
1.  **Construct Contour Tree:** Generate the contour tree for the scalar field. This involves identifying critical points and tracing the evolution of level sets [S1, unofficial].
2.  **Map Topology to Terrain:** Assign 3D coordinates to the nodes of the contour tree.
    *   The vertical axis ($z$) typically corresponds to the scalar value (isovalue) of the critical point.
    *   The horizontal axes ($x, y$) are determined by a layout algorithm. Weber et al. [45] propose using established 2D visualization techniques to lay out the data, though specific coordinate formulas are not detailed in the source [S1, unofficial].
3.  **Render Terrain:** Render the resulting 3D surface.
    *   **Valleys:** Represent topological events where a single contour splits into two or more parts (saddle points in the tree) [S1, unofficial].
    *   **Peaks:** Represent events where two or more contours merge (maxima in the tree) [S1, unofficial].

**Parameters:**
*   **Layout Algorithm:** The specific method for determining $x, y$ coordinates is not fixed; the source notes it was expanded using various 2D visualization techniques [S1, unofficial].
*   **Dimensionality:** The method applies to contour trees of any number of dimensions, allowing exploration of high-dimensional datasets [S1, unofficial].

**Edge Cases:**
*   **High-Dimensional Data:** The method is designed to handle high-dimensional data sets, but the complexity of the layout increases with dimensionality [S1, unofficial].
*   **Topological Validity:** The source emphasizes verifying that topological structures generated are valid before rendering [S1, unofficial].

## Subtopics

### [Contour Tree](9312-contour-tree.md)
The underlying graph structure where valleys represent contour splits and peaks represent contour merges. It is restricted to simply connected domains without boundaries or topological holes, unlike the Reeb Graph.

## Operating characteristics
*   **Strengths:** Harnesses human natural ability to understand terrain structure, providing an easier-to-understand model of underlying data topology [S1, unofficial]. Applicable to high-dimensional data sets [S1, unofficial].
*   **Limitations:** Requires the construction of a valid contour tree, which can be computationally intensive for large datasets. The quality of interpretation depends heavily on the chosen 2D layout technique for the horizontal axes [S1, unofficial].
*   **Performance:** Not given in the sources.

## Background
Topological Landscape was proposed by Weber et al. [45] as an alternative approach to viewing contour information [S1, unofficial]. It was further expanded using different methods for laying out the data, primarily from established 2D visualization techniques [S1, unofficial]. The method is part of the broader field of topological visualization techniques for volume multifield data [S1, unofficial].

## Sources
[S1] Chapter Topological Visualisation Techniques for Volume Multifield Data, https://archive.org/details/oapen-20.500.12657-49310, fetched 2026-10-27 (unofficial)
