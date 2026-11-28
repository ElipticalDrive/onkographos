# Reeb Graph


## Overview
The Reeb graph is a topological data structure that generalizes the contour tree to compute the topology of scalar data on domains that are not simply connected (contain loops) or lack a boundary [S1, unofficial]. It solves the problem of capturing global topology in complex geometries where standard contour trees fail [S1, unofficial]. It sets itself apart from the [Contour Tree](9312-contour-tree.md) by explicitly handling domains with loops or missing boundaries, and from the [Joint Contour Net](9316-joint-contour-net.md) by operating on single-variable scalar fields rather than multivariate data [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar field defined on a simplicial mesh [S1, unofficial]. The algorithm uses the 2-skeleton of the input volume to define relationships between vertices, edges, and triangles, a restriction that applies to scalar fields in any dimension [S1, unofficial].
*   **Output:** A graph representing the topology of the scalar field's level sets [S1, unofficial].

**Data Structures:**
The method relies on a simplicial mesh representation (vertices, edges, triangles) [S1, unofficial].

**Algorithm Steps:**
Reeb graph algorithms are split into two groups: sweep-based methods and split-and-compute methods [S1, unofficial].

1. **Sweep-based Methods:** These methods require the maintenance of level sets as the scalar value sweeps through the domain [S1, unofficial]. Doraiswamy and Natarajan proposed an approach that explicitly maintains level sets, eliminating the need for loop-surgery pre-processing and generalizing to higher-dimensional inputs [S1, unofficial].
2. **Split-and-Compute Methods:** These methods collapse the problem to a contour tree computation [S1, unofficial].
    * **Loop Surgery (Tierny et al.):** Perform symbolic cuts during the merge tree step to break loops, then stitch them back together to retrieve a topologically correct Reeb graph [S1, unofficial].
    * **Optimized Split-and-Compute (Doraiswamy and Natarajan):** Use the join tree to identify potential loops, segment the domain into multiple loop-free contour trees instead of a single one, and compute these regions in parallel [S1, unofficial].
3. **Streaming Algorithm (Pascucci et al.):** An online algorithm that processes data points as they arrive, continually updating the output [S1, unofficial]. This limits peak memory usage but has higher time complexity [S1, unofficial].
4. **Random Collapse (Harvey et al.):** Randomly collapses triangles for arbitrary 2-skeletons to improve running time [S1, unofficial].

**Formulas and Definitions:**
*   Streaming algorithm complexity: $O(n^2)$, where $n$ is the number of triangles [S1, unofficial].
*   Random collapse complexity: $O(m \log m)$, where $m$ is the number of triangles [S1, unofficial].

**Parameters:**
*   **Input Representation:** The input must be represented on a simplicial mesh [S1, unofficial].
*   **Dimension:** Applicable to models of any dimension, provided the 2-skeleton restriction is met [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Genus Sensitivity:** Computations are sensitive to the topological genus of the input domain (e.g., sphere vs. torus) [S1, unofficial]. The number of loops from the Morse function extends algorithm complexity [S1, unofficial].
*   **Critical Point Precision:** Variations in neighbor definition can slightly alter critical vertex locations without changing the graph structure [S1, unofficial].
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
*   **Strengths:** Correctly handles non-simply connected domains and domains without boundaries, which the contour tree cannot [S1, unofficial]. Supports online/streaming computation [S1, unofficial]. Generalizes to higher dimensions [S1, unofficial].
*   **Limitations:** Computation is sensitive to topological genus; more loops increase complexity [S1, unofficial]. Requires a simplicial mesh representation [S1, unofficial].
*   **When to use:** Use when the scalar field is defined on a domain with loops or no boundary, and single-variable topology is sufficient [S1, unofficial]. Use when online/streaming computation is required [S1, unofficial].
*   **When not to use:** Use the [Contour Tree](9312-contour-tree.md) if the domain is simply connected with a boundary, as it is simpler [G]. Use [Joint Contour Net](9316-joint-contour-net.md) or [Multi-Dimensional Reeb Graph](9382-multi-dimensional-reeb-graph.md) if the data is multivariate [G].

## Background
The Reeb graph was first used for encoding topological features for visualization purposes by Shinagawa and Kunii, who applied it to represent objects from computerized-tomography (CT) sources [S1, unofficial]. The sensitivity of Reeb graph computations to the topological genus of the input domain was first demonstrated by McLaughlin [S1, unofficial]. Pascucci et al. later provided an online streaming algorithm [S1, unofficial]. Tierny et al. introduced the "loop-surgery" technique to address genus sensitivity [S1, unofficial]. Doraiswamy and Natarajan proposed alternative approaches, including explicit level set maintenance and optimized split-and-compute techniques using join trees [S1, unofficial]. The Reeb graph has also been used to assist in the design of transfer functions in volume visualization by assigning opacity based on how many objects were obscured by nested surfaces and their proximity to the edge of a scalar field [S1, unofficial].

## Sources
[S1] Chapter Topological Visualisation Techniques for Volume Multifield Data, https://archive.org/details/oapen-20.500.12657-49310, fetched 2026-10-27 (unofficial)
