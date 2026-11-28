# Curvature

Also called: Curvatura

## Overview
Curvature is a feature-based classification metric that measures how much a data surface deviates from a straight aspect by analyzing local shape contours [S1, unofficial]. It is primarily used in non-photorealistic visualization to distinguish transitions in tissue or surface geometry [S1, unofficial]. Unlike methods relying solely on intensity or gradient magnitude, curvature utilizes second-order derivative information to identify structural features [S1, unofficial].

## How it works
**Inputs:** A scalar volume dataset or a triangular mesh representation of the data [S1, unofficial][S2].
**Outputs:** A scalar curvature value (Mean or Gaussian) or a Boolean classification (Convex/Concave) per voxel or vertex [S1, unofficial][S2].

### Continuous Curvature (Volume)
For volumetric data, curvature is derived from the second directional derivative of the intensity function $f$ [S1, unofficial]. The most precise approach uses the Hessian matrix $H$, where $f''$ represents the second directional derivative [S1, unofficial]. An alternative, lower-cost approximation uses the Laplacian operator $\nabla^2 f$ [S1, unofficial]. From the principal curvatures (maximum $k_1$ and minimum $k_2$), specific curvature types are computed [S1, unofficial]: Mean Curvature ($H$) is the average of principal curvatures [S1, unofficial]. Gaussian Curvature ($K$) is the product of principal curvatures [S1, unofficial].

### Discrete Mean Curvature Classification (Mesh)
For mesh-based data, a Boolean classification distinguishes convex regions (gyri) from concave regions (sulci) using the direction of the discrete mean curvature vector relative to the surface normal [S2]. This method uses only the sign and direction of the operator, not its magnitude [S2].

**Data Structures:**
*   **Vertex:** $x_i$ [S2].
*   **1-Ring Neighbors:** Set of vertices $\{x_j\}$ connected to $x_i$ by an edge $e_{ij}$ [S2].
*   **Angles:** $\beta_j$ is the angle opposite the edge $e_{ij}$ in the adjacent faces sharing that edge [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def classify_curvature(mesh):
    for vertex in mesh.vertices:
        normal = compute_normal(vertex)
        curvature_dir = Vector(0, 0, 0)
        for neighbor in vertex.one_ring_neighbors:
            cot_weight = cotangent_opposite_angle(vertex, neighbor)
            diff = neighbor.position - vertex.position
            curvature_dir += cot_weight * diff
        dot_product = dot(curvature_dir, normal)
        if dot_product > 0:
            vertex.classification = "convex"
        elif dot_product < 0:
            vertex.classification = "concave"
```

**Steps:**
1.  Compute the discrete normal vector $n_i$ at vertex $x_i$ [S2].
2.  Compute the simplified direction operator $K_{dir}(x_i)$ as a weighted sum of difference vectors to 1-ring neighbors [S2]:
    $$ K_{dir}(x_i) = \sum_{j=1}^{N_i} \cot(\beta_j) (x_j - x_i) $$
    Where $N_i$ is the number of neighbors, and $\beta_j$ is the angle opposite the edge connecting $x_i$ and $x_j$ [S2].
3.  Classify the vertex based on the dot product with the normal [S2]:
    *   **Convex:** If $K_{dir}(x_i) \cdot n_i > 0$ (vectors point in roughly the same direction) [S2].
    *   **Concave:** If $K_{dir}(x_i) \cdot n_i < 0$ (vectors point in roughly opposite directions) [S2].

**Pitfalls:**
*   The Laplacian approximation is computationally cheaper but yields less precise results than the full Hessian approach [S1, unofficial].
*   Boundary conditions and degenerate triangles (zero area) can destabilize the cotangent weights [G].

## Subtopics

### Discrete Mean Curvature Classification
A Boolean classification operator that distinguishes convex regions (gyri) from concave regions (sulci) based on the direction of the discrete mean curvature vector relative to the surface normal [S2]. Unlike methods that use the length of the curvature operator for scalar values, this method relies exclusively on the sign of the dot product between the curvature direction vector and the vertex normal [S2]. It is chosen when the goal is topological segmentation of surface features (e.g., brain cortex folding) rather than continuous shading [S2].

## Operating characteristics
*   **Complexity:** The Hessian-based approach is more computationally expensive than the Laplacian approximation [S1, unofficial]. The discrete mesh method requires iterating over 1-ring neighborhoods, making it linear with respect to the number of edges [S2].
*   **Strengths:** Effective for non-photorealistic rendering and structural feature detection (ridges/valleys) [S1, unofficial]. The discrete directional method robustly separates convex/concave topology without sensitivity to magnitude scaling [S2].
*   **Limitations:** The Laplacian approximation sacrifices precision [S1, unofficial]. Curvature is sensitive to noise in the input data, often requiring pre-smoothing [G].

## Background
Curvature-based transfer functions were developed to enhance the visualization of complex structures where intensity alone is insufficient [S1, unofficial]. The concept is rooted in differential geometry, utilizing the Hessian matrix or Laplacian to approximate second derivatives [S1, unofficial]. The discrete mesh operator described in [S2] is based on approximation schemes derived by Meyer et al. [S2].

## Sources
[S1] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
[S2] Hierarchical Isosurface Segmentation Based on Discrete Curvature, https://inria.hal.science/inria-00537470v1, fetched 2026-10-26
