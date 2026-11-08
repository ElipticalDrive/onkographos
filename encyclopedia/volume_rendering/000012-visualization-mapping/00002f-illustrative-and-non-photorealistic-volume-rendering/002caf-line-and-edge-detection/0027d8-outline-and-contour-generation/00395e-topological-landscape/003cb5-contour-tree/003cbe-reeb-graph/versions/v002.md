# Reeb Graph

Also called: Reeb Space (for multivariate generalizations, though distinct in discrete implementation)

## Overview
The Reeb graph is a topological data structure that generalizes the contour tree to handle scalar fields defined on domains that are not simply connected (i.e., domains containing loops or lacking a boundary) [S1, unofficial]. While the contour tree tracks the evolution of connected components of level sets, the Reeb graph tracks the evolution of connected components of level sets modulo homotopy, effectively identifying points in the domain that can be connected by a path of constant scalar value [S1, unofficial]. It solves the problem of capturing the global topology of scalar data in complex geometries, such as toroidal volumes or data with internal voids, where standard contour trees fail or produce ambiguous results [S1, unofficial]. It sets itself apart from the [Contour Tree](9312-contour-tree.md) by explicitly handling genus (loops) in the input domain, and from the [Joint Contour Net](9316-joint-contour-net.md) by operating on single-variable scalar fields rather than multivariate data.

## How it works
The Reeb graph is constructed by analyzing the connectivity of level sets of a scalar function $f: M \to \mathbb{R}$ defined on a domain $M$. The algorithm typically operates on a discretized representation of the domain.

**Inputs and Outputs:**
*   **Input:** A scalar field $f$ defined on a simplicial mesh (specifically a 2-skeleton for 3D volumes, or generally an $(n-1)$-skeleton for $n$-dimensional data) [S1, unofficial]. The mesh consists of vertices, edges, and triangles (or higher-dimensional simplices).
*   **Output:** A graph where nodes represent critical points (minima, maxima, saddles) and connected components of regular level sets, and edges represent the evolution of these components between critical values [S1, unofficial].

**Data Structures:**
The implementation relies on the simplicial mesh structure. For an $n$-dimensional input, the algorithm requires the $(n-1)$-skeleton of the triangulation [S1, unofficial]. In 3D, this is the 2-skeleton (vertices, edges, and triangular faces) [S1, unofficial]. The graph itself is typically stored as an adjacency list or a merge/join tree structure that is subsequently modified to handle loops.

**Algorithm Steps:**
Reeb graph algorithms are broadly categorized into two groups: sweep-based methods and split-and-compute methods [S1, unofficial].

1.  **Sweep-based Methods:**
    *   These methods maintain the explicit structure of level sets as the scalar value sweeps through the domain [S1, unofficial].
    *   As the sweep line (or plane/hyperplane) moves, it updates the connectivity of the level sets.
    *   This approach explicitly maintains level sets, which eliminates the need for pre-processing steps like loop surgery but can be computationally expensive [S1, unofficial].
    *   Doraiswamy and Natarajan proposed an approach that explicitly maintains level sets, generalizing to higher-dimensional inputs [S1, unofficial].

2.  **Split-and-Compute Methods (via Contour Tree):**
    *   These methods reduce the Reeb graph computation to a contour tree computation by temporarily simplifying the topology [S1, unofficial].
    *   **Loop Surgery (Tierny et al.):**
        1.  Perform symbolic cuts on the domain to break all loops, converting the non-simply connected domain into a simply connected one [S1, unofficial].
        2.  Compute the contour tree (specifically the merge tree) of the cut domain [S1, unofficial].
        3.  Stitch the symbolic cuts back together to retrieve the topologically correct Reeb graph [S1, unofficial].
    *   **Optimized Split-and-Compute (Doraiswamy and Natarajan):**
        1.  Use the join tree of the data to identify potential loops [S1, unofficial].
        2.  Segment the domain into multiple loop-free contour trees instead of a single global one [S1, unofficial].
        3.  Compute the contour trees for these segments in parallel [S1, unofficial].
        4.  Reconstruct the Reeb graph by merging the results, accounting for the identified loops [S1, unofficial].

3.  **Streaming Algorithm (Pascucci et al.):**
    *   This is an online algorithm that processes data points as they arrive [S1, unofficial].
    *   The output is continually updated as additional data points are added [S1, unofficial].
    *   This approach limits peak memory usage, making it suitable for large datasets, but has higher time complexity [S1, unofficial].

4.  **Random Collapse (Harvey et al.):**
    *   For arbitrary 2-skeletons, triangles are randomly collapsed to improve running time [S1, unofficial].
    *   This method aims to reduce the complexity to $O(m \log m)$, where $m$ is the number of triangles [S1, unofficial].

**Formulas and Definitions:**
*   Let $f: M \to \mathbb{R}$ be the scalar function.
*   The Reeb graph $R_f$ is the quotient space of $M$ where two points $x, y \in M$ are equivalent if they belong to the same connected component of the level set $f^{-1}(c)$ for some $c \in \mathbb{R}$ [S1, unofficial].
*   Complexity of the streaming algorithm: $O(n^2)$, where $n$ is the number of triangles [S1, unofficial].

**Parameters:**
*   **Input Representation:** The method requires the input to be represented on a simplicial mesh [S1, unofficial].
*   **Dimension:** The method applies to models of any dimension, provided the appropriate skeleton is used [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Genus Sensitivity:** Reeb graph computations are sensitive to the topological genus of the input domain (e.g., sphere vs. torus) [S1, unofficial]. The number of loops in the data extends algorithm complexity [S1, unofficial].
*   **Critical Point Precision:** Variations in neighbor definition can slightly alter the location of critical vertices without changing the overall graph structure [S1, unofficial].
*   **Stability:** Output remains fixed provided the simplicial subdivision does not change [S1, unofficial].

## Subtopics

### [Joint Contour Net](9316-joint-contour-net.md)
The Joint Contour Net (JCN) is a discrete graph-based representation of the Reeb space for multivariate scalar data, approximating the topology of $n$ variables defined over an $m$-dimensional domain. It solves the problem of visualizing and analyzing the topological relationships between multiple scalar fields by partitioning the domain into connected regions called joint contour slabs. Unlike the [Reeb Graph](9315-reeb-graph.md), which handles single-variable topology, the JCN extends this to multivariate inputs; it differs from the [Multi-Dimensional Reeb Graph](9382-multi-dimensional-reeb-graph.md) by using a quantization-based slab construction rather than critical node extraction.

### [Multi-Dimensional Reeb Graph](9382-multi-dimensional-reeb-graph.md)
The Multi-Dimensional Reeb Graph (MDRG) is a hierarchical data structure used to represent the topology of multivariate scalar fields by recursively storing joint contours. It solves the problem of capturing complex topological relationships in domains with multiple independent functions (e.g., pressure, temperature) where standard univariate methods fail. It sets itself apart from the [Joint Contour Net](9316-joint-contour-net.md) by providing a recursive, tiered representation of the Reeb space, which serves as an intermediate structure for extracting the Jacobi structure and the Reeb skeleton.

### [Handle Size Measurement via Nonseparating Cuts](9825-handle-size-measurement-via-nonseparating-cuts.md)
This method measures the geometric size of topological handles in a scalar field by computing two specific nonseparating loops: the Reeb loop and the cross loop. It solves the problem of quantifying handle prominence to guide topology simplification, ensuring that geometric changes to the volume are minimized. It differs from sibling methods like [Joint Contour Net](9316-joint-contour-net.md) by explicitly constructing these minimal loops on the isosurface to define a handle size metric, rather than relying solely on multi-dimensional topological descriptors.

## Operating characteristics
*   **Time Complexity:** The streaming algorithm by Pascucci et al. has $O(n^2)$ complexity, where $n$ is the number of triangles [S1, unofficial]. This is optimal for 2D inputs but performs less favorably for higher-dimensional inputs [S1, unofficial]. Harvey et al. propose a method with $O(m \log m)$ complexity for arbitrary 2-skeletons [S1, unofficial].
*   **Memory Complexity:** The streaming nature of the Pascucci et al. algorithm limits peak memory usage [S1, unofficial].
*   **Strengths:** It correctly handles non-simply connected domains (genus > 0) and domains without boundaries, which the contour tree cannot [S1, unofficial]. It can be computed online (streaming) [S1, unofficial]. It generalizes to higher dimensions [S1, unofficial].
*   **Limitations:** Computation is sensitive to the topological genus; more loops increase complexity [S1, unofficial]. Requires a simplicial mesh representation [S1, unofficial].
*   **When to use:** Use when the scalar field is defined on a domain with loops (e.g., torus) or no boundary, and single-variable topology is sufficient. Use when online/streaming computation is required.
*   **When not to use:** Use the [Contour Tree](9312-contour-tree.md) if the domain is simply connected and boundary exists, as it is simpler. Use [Joint Contour Net](9316-joint-contour-net.md) or [Multi-Dimensional Reeb Graph](9382-multi-dimensional-reeb-graph.md) if the data is multivariate.

## Background
The Reeb graph was first used for encoding topological features for visualization purposes by Shinagawa and Kunii [S1, unofficial]. They used the structure to represent objects obtained from computerized-tomography (CT) sources [S1, unofficial]. The sensitivity of Reeb graph computations to the topological genus of the input domain was first demonstrated by McLaughlin [S1, unofficial]. Pascucci et al. later provided an online streaming algorithm [S1, unofficial]. Tierny et al. introduced the "loop-surgery" technique to address genus sensitivity [S1, unofficial]. Doraiswamy and Natarajan proposed alternative approaches, including explicit level set maintenance and optimized split-and-compute techniques using join trees [S1, unofficial]. The Reeb graph has also been used to assist in the design of transfer functions in volume visualization by assigning opacity based on how many objects were obscured by nested surfaces and their proximity to the edge of a scalar field [S1, unofficial].

## Sources
[S1] Chapter Topological Visualisation Techniques for Volume Multifield Data, https://archive.org/details/oapen-20.500.12657-49310, fetched 2026-10-27 (unofficial)
