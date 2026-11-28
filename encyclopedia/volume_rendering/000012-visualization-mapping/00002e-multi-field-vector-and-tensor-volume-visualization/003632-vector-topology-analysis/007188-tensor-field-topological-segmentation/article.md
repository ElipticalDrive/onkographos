# Tensor Field Topological Segmentation

Also called: Eigenvector-based Topological Segmentation, Tensor Field Topology, Topological Skeleton Segmentation

## Overview
Tensor Field Topological Segmentation decomposes a symmetric second-order tensor field domain into cells based on its topological skeleton, formed by separatrices and degenerate points, followed by adaptive refinement based on eigenvalue similarity [S1, book][S2, unofficial]. It differs from standard vector field topology by handling the ambiguity of eigenvector directions at degenerate points, where every vector is an eigenvector, allowing trajectories to cross [S2, unofficial]. This method builds on [Vector Topology Analysis](7694-vector-topology-analysis.md) but extends it to handle the orthogonal eigenvector fields inherent in symmetric tensors.

## How it works
**Inputs:** A 2D symmetric second-order tensor field defined on a triangular mesh.
**Outputs:** A set of curvilinear cells representing the segmentation.

**Data Structures:**
*   **Tensor Field:** $T(x,y)$, a $2 \times 2$ symmetric matrix at each vertex.
*   **Degenerate Points:** Vertices or locations where eigenvalues $\lambda_1 = \lambda_2$.
*   **Separatrices:** Polylines representing trajectories of major (red) and minor (blue) eigenvector fields connecting degenerate points.
*   **Cells:** Defined by vertices which are either red-blue intersections, degenerate points, or boundary intersections.

**Steps:**

1.  **Extract Topology:**
    1.  **Locate Degenerate Points:** Identify points where both eigenvalues of the tensor are equal [S2, unofficial]. At these points, the eigenvector field is undefined (every vector is an eigenvector) [S2, unofficial].
    2.  **Determine Radial Directions:** For each degenerate point, determine the radial directions (separatrix entry/exit directions) [S1, book].
    3.  **Compute Separatrices:** Integrate the major and minor eigenvector fields using 4th order Runge-Kutta integration, triangle-wise [S1, book].
        *   **Termination Conditions:** Stop integration if:
            (a) The separatrix leaves the domain [S1, book].
            (b) The separatrix gets close to a degenerate point, line, or triangle [S1, book].
            (c) The separatrix describes a circle or spiral and passes itself closely in the parallel integration direction [S1, book].
        *   Store separatrices as polylines [S1, book].

2.  **Generate Initial Cells:**
    1.  **Intersect Separatrices:** Compute intersections between red (major eigenvector) and blue (minor eigenvector) separatrices [S1, book].
        *   *Optimization:* Track separatrices per triangle; only consider triangles containing at least one red and one blue line [S1, book].
    2.  **Define Vertices:** Cell vertices are:
        *   Red-blue intersection points [S1, book].
        *   Degenerate points [S1, book].
        *   Intersections of tensorlines with the domain boundary [S1, book].
    3.  **Form Cells:** Connect vertices to form curvilinear cells. Cells may lack degenerate points internally [S1, book].

3.  **Adaptively Refine Cells:**
    1.  **Coarsening:** Merge adjacent cells that do not exhibit enough structural information on their own [S1, book].
    2.  **Subdivision:** Split cells that exceed defined criteria of similarity with respect to eigenvalue behavior by introducing new tensorlines [S1, book].
    3.  **Hanging Node Management:** Use a growing strategy to merge or subdivide consecutive cells to minimize hanging nodes caused by divergence/convergence of tensor lines [S1, book].

**Formulas:**
*   **Degeneracy Condition:** $\lambda_1 = \lambda_2$ [S2, unofficial].
*   **Orthogonality:** Eigenvectors $v_1, v_2$ are orthogonal ($v_1 \cdot v_2 = 0$) except at degenerate points [S2, unofficial].

**Parameters:**
*   Integration order: 4th order Runge-Kutta [S1, book].
*   Similarity criteria: Based on scalar invariants derived from eigenvalues [S1, book].

**Edge Cases and Pitfalls:**
*   **Degenerate Points:** These are the only points where eigenvector trajectories can cross because eigenvectors are otherwise orthogonal [S2, unofficial].
*   **Hanging Nodes:** Adaptive segmentation inevitably causes hanging nodes; a growing strategy is required to manage them [S1, book].
*   **Interpolation:** Eigenvector-based interpolation may lead to rare configurations of degenerate points compared to component-based interpolation [S1, book].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Captures global features of the tensor field structure; handles the unique topology of symmetric tensors [S2, unofficial].
*   **Limitations:** Requires careful handling of degenerate points and hanging nodes; adaptive refinement can be complex to implement [S1, book].
*   **When to use:** When analyzing the structural organization of symmetric tensor fields, such as in Diffusion Tensor Imaging (DTI) [S2, unofficial].

## Background
The topological representation of symmetric second-order tensor fields was introduced by Lambertus Hesselink and Thierry Delmarcelle in 1994 [S2, unofficial]. The method is particularly relevant for visualizing neuroanatomy from DTI data, where it helps interpret water diffusion structures in the brain [S2, unofficial].

## Sources
[S1] DFU, Volume 2, Scientific Visualization: Interactions, Features, Metaphors, Complete Volume, https://doi.org/10.4230/dfu.vol2.sciviz.2011, fetched 2026-10-26 (book)
[S2] PDF 716Handout7_TensorIcons - Auckland, https://www.cs.auckland.ac.nz/courses/compsci716s2t/lectures/716Handout7_TensorIcons_1up.pdf, fetched 2026-10-28 (unofficial)
