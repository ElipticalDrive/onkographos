# Interval Volume Decomposer

Also called: IV decomposition order extraction, Level-Set Graph Traversal for IV Decomposition, Level-Set Graph Traversal for IV Decomposition Order, Systematic traversal of level-set graph, Topological Volume Skeletonization

## Overview
The Interval Volume Decomposer is an algorithm that decomposes a scalar volume into feature subvolumes, specifically interval volumes (IVs), by tracking the topological transitions of isosurfaces [S1, book]. It constructs a level-set graph representing the topological skeleton of the volume and enables systematic decomposition from the exterior to the interior by traversing this graph [S1, book]. It is distinguished from sibling methods by its explicit construction of a topological graph to manage inclusion relationships, relying on specific component algorithms for component counting, genus identification, and clustering [S1, book].

## How it works
**Inputs and Outputs:**
- Input: A scalar field volume $V$.
- Output: A set of decomposed interval volumes (IVs) with their inclusion hierarchy and decomposition order.

**Data Structures:**
- **Level-Set Graph:** A graph where nodes represent critical points of the scalar field and links represent interval volumes.
    - **Nodes:** Represent critical points where isosurface components undergo topological evolution. Classified by index (number of negative eigenvalues of the Hessian matrix): $C_3$ (maxima), $C_2$ (saddles), $C_1$ (saddles), and $C_0$ (minima) [S1, book].
    - **Links:** Each link corresponds to a feature subvolume, specifically an interval volume bounded by critical isosurfaces [S1, book].

**Algorithm Steps:**
1.  **Construct the Level-Set Graph:**
    The algorithm tracks the topological transitions of an isosurface according to the scalar field to build the level-set graph [S1, book]. This step combines three specific component algorithms [S1, book]:
    -   Use the algorithm of Carr et al. [6] to track the number of isosurface connected components [S1, book].
    -   Use the algorithm of Pascucci et al. [22] to identify the genus of each isosurface component [S1, book].
    -   Use the algorithm of Takahashi et al. [31] for IV clustering [S1, book].
    *(Note: The specific implementation details of these three component algorithms are not provided in the source text; they are referenced as external dependencies [S1, book].)*

2.  **Identify Interval Volumes:**
    Identify each link of the constructed level-set graph with a feature subvolume, which is an interval volume (IV) bounded by critical isosurfaces [S1, book]. An IV is defined as a subvolume corresponding to a specific range of scalar field values [S1, book].

3.  **Retrieve Inclusion Relationships:**
    Determine the inclusion relationships of the decomposed interval volumes by systematically traversing the level-set graph [S1, book]. The traversal starts from the end node corresponding to the exterior surface of the volume [S1, book]. This traversal allows the system to locate how IVs are nested within one another [S1, book].

4.  **Decompose Volume:**
    Peel the decomposed interval volumes from outside to inside based on the retrieved inclusion relationships [S1, book]. This yields the final set of feature subvolumes [S1, book].

**Parameters:**
-   The method relies on the integration of three external algorithms: Carr et al. [6], Pascucci et al. [22], and Takahashi et al. [31] [S1, book]. No internal tuning parameters are specified in the source [S1, book].

**Edge Cases and Pitfalls:**
-   The source notes that isosurface transitions at $C_2$ and $C_1$ saddles are classified into four different types to account for isosurface embeddings in 3D space [S1, book]. Implementation must correctly handle these transition types to maintain topological accuracy [S1, book].
-   Precision in identifying critical points (maxima, minima, saddles) is crucial for correct graph construction [S1, book].

## Subtopics

### Level-Set Graph Traversal for IV Decomposition
The parent method that provides the traversal framework. The Interval Volume Decomposer is the specific implementation/interface that performs the decomposition using this traversal. [Level-Set Graph Traversal for IV Decomposition](19443-level-set-graph-traversal-for-iv-decomposition.md)

## Operating characteristics
-   **Time and Memory Complexity:** Not given in the sources.
-   **Measured Performance:** Not given in the sources.
-   **Strengths:** Enables systematic decomposition from outside to inside; provides a topological skeleton of the volume; facilitates optimal viewpoint selection for 2D projection by arranging IVs [S1, book].
-   **Limitations:** Depends on the accuracy and performance of three external component algorithms (Carr, Pascucci, Takahashi) [S1, book].
-   **When to use:** When the goal is to decompose a volume into topologically meaningful feature subvolumes (IVs) for analysis or previewing inner structures [S1, book].
-   **When not to use:** When simple voxel-based segmentation is sufficient and topological transitions are not required [S1, book].

## Background
The Interval Volume Decomposer was implemented as an interface in a study by the authors of [S1, book], previously referred to as "interval volume decomposer [28]" [S1, book]. It was developed to consistently analyze and preview the inner structures of volumes by extracting feature subvolumes [S1, book]. The concept of Interval Volumes (IVs) was formulated by Fujishiro et al. [10] and Guo [13] as a generalization of the isosurface [S1, book]. The level-set graph construction algorithm, called "topological volume skeletonization," was developed by Takahashi et al. [31] [S1, book]. The method was also used in the formulation of viewpoint entropy for volumes [29].

## Sources
[S1] Previewing Volume Decomposition Through Optimal Viewpoints, https://doi.org/10.4230/dfu.vol2.sciviz.2011.346, fetched 2026-10-26 (book)
