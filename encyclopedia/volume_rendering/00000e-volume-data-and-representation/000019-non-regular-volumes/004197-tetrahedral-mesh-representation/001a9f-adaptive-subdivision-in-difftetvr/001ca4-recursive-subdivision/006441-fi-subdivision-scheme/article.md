# fi-subdivision scheme

Also called: 1/2-subdivision, 4-1/2 subdivision, fourth-root-of-two subdivision, half-subdivision

## Overview
The fi-subdivision scheme (also known as 1/2-subdivision) is a regular hierarchical data organization for $n$-dimensional grids that doubles the overall number of vertices in each subdivision step [S1, unofficial]. It solves the problem of achieving fine granularity and high adaptivity in multiresolution representations, particularly for time-varying volume data, by requiring fewer vertices than octrees to satisfy approximation bounds [S1, unofficial]. Unlike octree refinement, which grows the vertex count by a factor of $2^n$ in every dimension, fi-subdivision provides a growth factor of 2, making it more efficient for spatial dimensions [S1, unofficial].

## How it works
The method consists of a splitting step (topological refinement) and, optionally, an averaging step (value adjustment). The scheme is defined for arbitrary dimension $n$, with specific implementations detailed for $n=2$ (quadrilaterals) and $n=4$ (hypercuboids for space-time).

### Inputs and Outputs
*   **Input:** A grid element (quadrilateral in 2D, hypercuboid in 4D) with associated vertex positions and function values.
*   **Output:** A refined set of elements with new vertices inserted and connectivity updated.

### Data Structures
The implementation relies on a mesh representation where vertices and elements are tracked.
*   **Vertices:** Store position $\mathbf{v}$ and scalar function value $f$.
*   **Elements:** Store connectivity (list of vertex indices).
*   **Centroid:** A temporary or permanent vertex inserted at the geometric center of the parent element.

### Numbered Steps

#### 1. Splitting Step (Topological Refinement)
This step inserts new vertices and adjusts connectivity. The specific procedure depends on the dimension.

**For $n=2$ (Quadrilateral $Q$):**
1.  Compute the centroid $c$ of the quadrilateral $Q$. The centroid is the average of its four vertices $\mathbf{v}_i$:
    $$ c = \frac{1}{4} \sum_{i=1}^{4} \mathbf{v}_i $$
2.  Insert $c$ as a new vertex.
3.  Connect $c$ to all four vertices of $Q$.
4.  Remove the "old" edges of the mesh (the boundary of $Q$), except for edges that determine the global mesh/domain boundary. This results in four new triangular elements sharing vertex $c$.

**For $n=4$ (Hypercuboid for Space-Time):**
The refinement of a 4D hypercuboid (representing space + time) requires four sequential subdivision steps to achieve the topological result of one hexadectree refinement step [S1, unofficial]:
1.  **Step 1:** Insert the centroid of the hypercuboid. This can be interpreted as the centroid of a cuboid at an intermediate time step $t_2$ [S1, unofficial].
2.  **Step 2:** Insert the centroids of the eight cuboids within the original hypercuboid.
3.  **Step 3:** Insert the centers of the faces of these eight cuboids (or the original hypercuboid).
4.  **Step 4:** Insert the midpoints of the edges of the eight cuboids (or the original hypercuboid).
The resulting structure consists of 16 hypercuboids, topologically equivalent to one hexadectree refinement [S1, unofficial].

**Generalization to Arbitrary Dimension $n$:**
1.  Insert the centroid of the parent element.
2.  Adjust vertex connectivity to link the new centroid to the existing vertices of the parent element.

#### 2. Averaging Step (Value Adjustment)
This step updates the function values of the "old" vertices based on the newly inserted vertices. It is defined by the update rule [S1, unofficial]:
$$ v_{new} = a v_{old} + (1 - a) w $$
Where:
*   $v_{old}$ is the function value at an old vertex.
*   $v_{new}$ is the updated function value at the old vertex.
*   $a$ is a parameter (typically $0 \le a \le 1$).
*   $w$ is the centroid (average) of the function values of the adjacent new vertices surrounding the old vertex.

**Note on Parameter $a$:**
The optimal choice of $a$ is not analyzed for dimensions $> 2$ [S1, unofficial]. However, for rectilinear grids with uniform hypercuboids, the update rule does not affect the geometric positions of the vertices regardless of $a$, but only affects the dependent function values [S1, unofficial]. In the context of the primary source, B-spline wavelets replace fi-subdivision wavelets, rendering the parameter $a$ unnecessary for the final implementation [S1, unofficial].

### Pseudocode
The following pseudocode implements the splitting step for a 2D quadrilateral. The averaging step is omitted as the primary source replaces fi-subdivision wavelets with B-spline wavelets, making the averaging step with parameter $a$ redundant for the described application [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fi_subdivision_2d(quad):
    """
    Performs one fi-subdivision step on a single quadrilateral.
    
    Args:
        quad: An object or list containing 4 vertices [v0, v1, v2, v3].
              Each vertex has attributes: position (x, y), value (scalar).
    
    Returns:
        A list of 4 new triangular elements.
        Each triangle is defined by 3 vertex indices/objects.
    """
    # 1. Compute centroid
    # c = (v0 + v1 + v2 + v3) / 4
    c_pos = (quad[0].position + quad[1].position + 
             quad[2].position + quad[3].position) / 4.0
    c_val = (quad[0].value + quad[1].value + 
             quad[2].value + quad[3].value) / 4.0
    
    # Create new centroid vertex
    centroid = Vertex(position=c_pos, value=c_val)
    
    # 2. Connect centroid to all four vertices
    # The old edges are removed, creating 4 new triangles.
    # Triangle 1: v0, v1, c
    # Triangle 2: v1, v2, c
    # Triangle 3: v2, v3, c
    # Triangle 4: v3, v0, c
    
    tri1 = [quad[0], quad[1], centroid]
    tri2 = [quad[1], quad[2], centroid]
    tri3 = [quad[2], quad[3], centroid]
    tri4 = [quad[3], quad[0], centroid]
    
    return [tri1, tri2, tri3, tri4]
```

### Edge Cases and Pitfalls
*   **Boundary Edges:** When removing old edges, edges determining the mesh/domain boundary must be preserved to maintain domain integrity [S1, unofficial].
*   **Dimension > 2 Averaging:** The averaging step's parameter $a$ lacks analysis for $n > 2$. If used, $a$ should be chosen carefully or set to 0 if only geometric refinement is needed and values are handled by a separate wavelet scheme [S1, unofficial].
*   **Non-Rectilinear Grids:** The claim that $a$ does not affect vertex positions holds for rectilinear grids with uniform hypercuboids. For non-uniform grids, $a$ may influence geometry if the averaging step is applied to positions [S1, unofficial].

## Subtopics

### 1/2 subdivision hierarchy
A downsampling process that applies the 1/2-subdivision hierarchy to reduce data resolution. It can be optionally combined with linear B-spline wavelet encoding to create narrow masks for update rules, supporting out-of-core data exploration [S1, unofficial]. The downsampling ratio is defined as the original number of vertices divided by the number of vertices at the coarser resolution. In tests, a ratio of $2^{1/2}$ was used on 3D time-varying simulation data (Richtmyer-Meshkov instability, 274 time steps, $2048^2$ rectilinear grid per step) [S1, unofficial]. Two variants were compared: 1/2-subdivision hierarchy without wavelet encoding, and with linear B-spline wavelet encoding [S1, unofficial].

## Operating characteristics
*   **Complexity:** The scheme doubles the overall number of vertices per step, providing finer granularity than octrees (which grow by $2^n$) [S1, unofficial]. This requires fewer vertices to satisfy approximation bounds [S1, unofficial].
*   **Performance:** The method supports high adaptivity, especially in spatial dimensions [S1, unofficial].
*   **Limitations:** The literature provides no analysis of averaging steps for dimensions larger than two, limiting the ability to optimize the parameter $a$ for general $n$-dimensional cases [S1, unofficial].
*   **Use Cases:** Ideal for multiresolution data visualization of time-varying volume data where spatial and temporal scalability are required [S1, unofficial].

## Background
The fi-subdivision scheme was developed to overcome the topological constraints of regular structures like octrees, which are less adaptive [S1, unofficial]. The splitting step of the scheme goes back to Cohen and Daubechies for $n=2$ and Maubach for arbitrary $n$ [S1, unofficial]. The scheme is analogous to Kobbelt's subdivision for triangles [S1, unofficial]. It was applied to large-scale time-varying volume data, specifically hydrodynamics simulations of the Richtmyer-Meshkov instability [S1, unofficial].

## Sources
[S1] Hierarchical Representation of Time-Varying Volume Data with Fourth-Root-of-Two Subdivision and Quadrilinear B-Spline Wavelets, https://digital.library.unt.edu/ark:/67531/metadc1398031/?q=volume data compression rendering, fetched 2026-10-28 (unofficial)
