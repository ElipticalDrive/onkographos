# Advancing Front Isosurface Extraction with Guidance Field

Also called: Guidance Field Advancing Front, Schreiner et al. method

## Overview
This method extracts isosurfaces from implicit functions using an advancing front algorithm guided by a scalar field $g(\mathbf{x})$ that constrains edge lengths based on surface curvature and user parameters [S1]. It builds on the [Advancing Front Framework for Implicit Surfaces](819-advancing-front-framework-for-implicit-surfaces.md) by replacing heuristic edge sizing with a theoretically grounded guidance field that ensures robust front interference detection and well-shaped triangles [S1]. Unlike [Marching Cubes](Marching Cubes.md) or [Marching Tetrahedra](Marching Tetrahedra.md), which produce meshes requiring downstream processing, this method generates high-quality meshes directly, though it is computationally slower [S1, 963-multi-threaded-advancing-front-isosurface-extraction-with-guidance-field.md].

## How it works
The algorithm constructs a guidance field from initial surface samples to determine ideal edge lengths, culls redundant samples for efficiency, seeds the front, and iteratively advances the front by adding triangles or merging/splitting fronts based on interference checks [S1].

### Inputs and Outputs
- **Inputs**: Implicit function $f(\mathbf{x})$, its gradient $\nabla f$, its Hessian $H(f)$, isovalue $k$, accuracy parameter $\rho$, and well-shapedness parameter $\eta$ [S1].
- **Outputs**: A triangular mesh approximating the isosurface $f(\mathbf{x}) = k$ [S1].

### Data Structures
- **Sample Set $S$**: A set of points $\mathbf{s}$ on the surface, each associated with a maximum curvature $\kappa_{\max}(\mathbf{s})$ and ideal edge length $\iota(\mathbf{s})$ [S1].
- **Guidance Field $g$**: A scalar field defined as the lower envelope of cone functions $\tilde{g}_s$ for all $s \in S$ [S1].
- **Active Fronts**: A collection of front objects, where each front represents a boundary edge or sequence of edges on the mesh [S1].
- **KD-Tree**: A 4-dimensional KD-tree used to store samples for culling, where each node stores coordinates $[s_x, s_y, s_z, s_r]$, with $[s_x, s_y, s_z]$ being the sample location and $s_r$ being the height of the cone function $\tilde{g}_s$ at that location [S1].

### Formulas
1.  **Ideal Edge Length**:
    $$ \iota(s) = \frac{2 \sin(\rho/2)}{\kappa_{\max}(s)} $$
    where $\iota(s)$ is the ideal edge length at sample $s$, $\rho$ is the accuracy parameter, and $\kappa_{\max}(s)$ is the maximum absolute curvature at $s$ [S1].

2.  **Cone Function**:
    $$ \tilde{g}_s(x) = (1 - \eta^{-1}) \cdot |x - s| + \eta^{-1} \cdot \iota(s) $$
    where $\tilde{g}_s$ is the cone function induced by sample $s$, $x$ is a point in embedding space, $s$ is a sample point on the surface, and $\iota(s)$ is the ideal edge length at $s$ [S1].

3.  **Guidance Field**:
    $$ g(x) = \min_{s \in S} \tilde{g}_s(x) $$
    where $g(x)$ is the guidance field, $S$ is the set of sampled points, and $\tilde{g}_s$ is the cone function for sample $s$ [S1].

4.  **Projection Matrix**:
    $$ P = I - n n^T H $$
    where $P$ is the projection matrix onto the tangent plane, $I$ is the identity matrix, $n = \nabla f / |\nabla f|$ is the normalized gradient, and $H$ is the Hessian matrix of $f$ [S1]. Note: The source text contains typos in the Hessian matrix entries (e.g., $\partial^2 f / \partial x \partial x$ in off-diagonals), but the structure is $I - n n^T$ multiplied by the Hessian [S1].

5.  **Curvature Bound**:
    $$ \kappa_{\max} \leq \|P\| \cdot \|H\| \cdot \|P\| \cdot \|I/|\nabla f|\| $$
    where $\kappa_{\max}$ is the maximum curvature, $P$ is the projection matrix, $H$ is the Hessian, and $\|\cdot\|$ denotes the matrix norm (specifically Frobenius norm $\|M\|_2 = \sqrt{\sum m_{ij}^2}$) [S1].

### Parameters
- **$\rho$**: User-defined parameter controlling reconstruction accuracy. It determines the angle subtended by edges on the minimum osculating circle. Typical values: 0.2, 0.3, 0.5 [S1].
- **$\eta$**: User-defined parameter controlling the rate of triangle expansion (well-shapedness). It bounds the ratio of adjacent edge lengths: $|e_1|/|e_2| \leq \eta$ if $|e_1| > |e_2|$. Typical values: 2, 3, 5 [S1].

### Steps
1.  **Generate Samples**: Create a jittered random set of points inside cells neighboring the surface and project them onto the surface via gradient descent to form set $S$ [S1].
2.  **Compute Guidance Field**: Evaluate the lower envelope of cone functions $\tilde{g}_s$ for all $s \in S$ to form $g(x)$ [S1].
3.  **Cull Guidance Field**: Remove unnecessary samples from $S$ to minimize memory and computation time while preserving the properties of $g(x)$ [S1].
4.  **Seed Fronts**: Compute seed points (e.g., using efficient algorithms like [6]) and project them onto the surface using gradient descent to initialize the `Active` front set [S1].
5.  **Advance Front**: While `Active` is not empty:
    a. Get a front from `Active`.
    b. Check if it is OK to add a triangle using parameters $\rho$ and $\eta$.
    c. If OK, add the triangle to the front.
    d. If not OK, get the interfering front.
    e. If the interfering front is the same as the current front, split the front into two new fronts ($f_1, f_2$), remove the old front, and add the two new fronts to `Active`.
    f. If the interfering front is different, merge the current front and the interfering front into a new front, remove both from `Active`, and add the new front to `Active` [S1].
6.  **Create Triangles**: Determine the correct edge size from the guidance field $g$. Project the new point onto the surface using the dihedral angle of the edge as a free parameter and searching for the isovalue [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def TRIANGULATE(f, grad_f, hess_f, k, rho, eta):
    # Step 1: Generate initial samples
    S = []
    # Create jittered random points in cells neighboring the surface f(x) = k
    # Project each point onto the surface using gradient descent
    # Append projected points to S
    pass # See Section 4.1 of [S1] for sampling condition

    # Step 2: Compute guidance field
    # For each s in S, compute kappa_max(s) using P and H
    # Compute iota(s) = (2 * sin(rho/2)) / kappa_max(s)
    # Define cone function g_tilde_s(x) = (1 - 1/eta) * |x - s| + (1/eta) * iota(s)
    # g(x) is the lower envelope of all g_tilde_s
    pass

    # Step 3: Cull guidance field
    CULL(g, rho, eta)

    # Step 4: Seed the front
    Active = SEED_FRONTS(S) # See Section 3.2 of [S1]

    # Step 5: Advance front
    while len(Active) > 0:
        front = GET_ANY_FRONT(Active)
        if OK_TO_ADD_TRIANGLE(front, rho, eta):
            ADD_TRIANGLE_TO_FRONT(front, rho, eta)
        else:
            other = GET_INTERFERING_FRONT(front)
            if other == front:
                f1, f2 = SPLIT(front)
                REMOVE_FRONTS(Active, {front})
                ADD_FRONTS(Active, {f1, f2})
            else:
                new_front = MERGE(front, other)
                REMOVE_FRONTS(Active, {front, other})
                ADD_FRONTS(Active, {new_front})

def CULL(g, rho, eta):
    # Insert all points in the guidance field into a 4-dimensional kd-tree
    # Coordinates for each sample s are [sx, sy, sz, sr], where [sx, sy, sz] is location
    # and sr is the height of g_tilde_s at s
    tree = KD_TREE(g.samples)
    
    # Sort samples by maximum curvature (kappa_max) in descending order
    sorted_samples = SORT(g.samples, key=lambda s: s.kappa_max, reverse=True)
    
    for s_i in sorted_samples:
        if not s_i.marked:
            # Construct a cone c defined by s_i, rho, eta
            # Apex: [sx, sy, sz, sr], Axis: [0, 0, 0, 1], Angle: atan(eta / (eta - 1))
            c = CONE(s_i, rho, eta)
            
            # Query kd-tree for points inside cone c and mark them
            MarkIfInside(tree, c)
            
            # If all children of a kd-tree node are marked, mark the node
            pass # Pruning logic for KD-Tree

    # Discard all marked samples from g(x)
    g.samples = [s for s in g.samples if not s.marked]
```

### Edge Cases and Pitfalls
- **Front Interference**: If the test fails to detect front interference, two or more sheets of triangles may form over portions of the surface. The algorithm uses a robust front intersection test procedure proven by the guidance field properties [S1].
- **Insufficient Sampling**: If the guidance field is not sufficiently sampled, features smaller than the sampling density can be missed, and $g(x)$ may not bound $\iota(x)$, leading to triangles larger than allowed by curvature [S1].
- **Sharp Curvature Changes**: Standard adaptive advancing front algorithms fail when sharp changes in curvature lead to sharp changes in triangle size and poorly shaped triangles. The guidance field prevents this by bounding the gradient magnitude of $g$ [S1].
- **Critical Points**: The method assumes the isosurface does not pass through critical points of $f$ [S1].

## Subtopics

### [Multi-threaded Advancing Front Isosurface Extraction](963-multi-threaded-advancing-front-isosurface-extraction.md)
This method parallelizes the Advancing Front Isosurface Extraction with Guidance Field [947-advancing-front-isosurface-extraction-with-guidance-field.md] by distributing the work of projecting points onto the advancing fronts across a set of worker threads. Compared to Marching Cubes [Marching Cubes](Marching Cubes.md) or Marching Tetrahedra [Marching Tetrahedra](Marching Tetrahedra.md), the algorithm is slower but generates meshes requiring negligible or no downstream processing, unlike MC/MT results.

### Minimal Guidance Field Culling
A technique to reduce the number of samples in a guidance field by removing samples that are 'dominated' by others, where a sample $s_1$ dominates $s_2$ if the edge length required for curvature at $s_1$ is always smaller than that of $s_2$, ensuring the guidance field remains sufficient and Lipschitz-continuous [S1]. It inserts all points into a 4-dimensional kd-tree, sorts samples by maximum curvature in descending order, and iteratively marks samples inside cones defined by dominant samples, pruning branches when all children are marked [S1].

## Operating characteristics
- **Time Complexity**: The culling step typically removes a large percentage of sample points, resulting in significantly faster guidance field queries and total running time, despite the additional processing overhead [S1]. For example, culling can reduce running time from 27m05s to 7m27s for the Silicium dataset with $\rho=0.2$ [S1].
- **Memory Complexity**: Culling significantly reduces memory usage by discarding unnecessary samples [S1].
- **Performance**: The method is slower than [Marching Cubes](Marching Cubes.md) or [Marching Tetrahedra](Marching Tetrahedra.md) but produces high-quality meshes with negligible downstream processing needs [S1, 963-multi-threaded-advancing-front-isosurface-extraction-with-guidance-field.md].
- **Strengths**: Robust front interference detection, well-shaped triangles, and guaranteed feature capture with sufficient sampling [S1].
- **Limitations**: Requires sufficient sampling to avoid missing features; assumes smooth surfaces and no critical points on the isosurface [S1].
- **When to Use**: When high-quality, well-shaped meshes are required without downstream processing, and computational resources allow for the higher cost of the advancing front algorithm [S1].
- **When Not to Use**: When real-time performance is critical, or when the surface has sharp features that cannot be adequately captured by the sampling density [S1].

## Background
The method is based on the work of Schreiner et al. [45], who introduced a solid theoretical foundation for advancing front algorithms using a guidance field [S1]. It addresses issues such as sharp changes in curvature and robust front interference detection that hindered previous advancing front algorithms [S1]. The enhancements described in [S1] (sampling condition and culling procedure) were presented to guarantee correct reproduction of features and reduce memory usage [S1].

## Sources
[S1] High-Quality Extraction of Isosurfaces from Regular and Irregular Grids, https://doi.org/10.1109/tvcg.2006.149, fetched 2026-10-26
