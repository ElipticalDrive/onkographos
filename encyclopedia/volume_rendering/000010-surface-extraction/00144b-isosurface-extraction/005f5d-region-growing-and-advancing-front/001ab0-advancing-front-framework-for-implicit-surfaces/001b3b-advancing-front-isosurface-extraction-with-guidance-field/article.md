# Advancing Front Isosurface Extraction with Guidance Field

Also called: Guidance Field Advancing Front, Schreiner et al. method

## Overview
This method extracts isosurfaces from implicit functions using an advancing front algorithm guided by a scalar field $g(\mathbf{x})$ that constrains edge lengths based on surface curvature and user parameters [S1]. It solves the problems of front interference detection and poor triangle shaping inherent in standard adaptive advancing front algorithms by providing a theoretical foundation for edge size selection [S1]. Unlike its parent [Advancing Front Framework for Implicit Surfaces](819-advancing-front-framework-for-implicit-surfaces.md), which relies on intrinsic geometry heuristics, this method uses a guidance field to bound the Hausdorff distance and aspect ratio of triangles, ensuring robust front intersection tests [S1].

## How it works
**Inputs:** Implicit function $f(\mathbf{x})$, its gradient $\nabla f$, its Hessian $H(f)$, isovalue $k$, accuracy parameter $\rho$, and well-shapedness parameter $\eta$ [S1].
**Outputs:** A triangulated mesh approximating the isosurface $f(\mathbf{x}) = k$ [S1].

**Data Structures:**
*   **Sample Set $S$:** A set of 3D points on the surface.
*   **Guidance Field $g$:** A scalar field defined as the lower envelope of cone functions.
*   **Active Fronts:** A collection of front objects, where each front represents a boundary edge or sequence of edges on the mesh.
*   **4D KD-Tree:** Used for culling samples, storing points as $[s_x, s_y, s_z, s_r]$ where $s_r$ is the height of the cone function at the sample location [S1].

**Parameters:**
*   $\rho$: Controls reconstruction accuracy. Determines the angle subtended by edges on the minimum osculating circle. Typical values: 0.2, 0.3, 0.5 [S1].
*   $\eta$: Controls the rate of triangle expansion (well-shapedness). Bounds the ratio of adjacent edge lengths $|e_1|/|e_2| \le \eta$ if $|e_1| > |e_2|$. Typical values: 2, 3, 5 [S1].

**Steps:**

1.  **Generate Initial Samples:** Create a jittered random set of points inside cells neighboring the surface. Project these points onto the surface $f(\mathbf{x})=k$ using gradient descent [S1].
2.  **Compute Guidance Field:** For each sample $s \in S$, compute the ideal edge length $\iota(s)$ and the cone function $\tilde{g}_s(\mathbf{x})$. The guidance field is the lower envelope:
    $$ \iota(s) = \frac{2 \sin(\rho/2)}{\kappa_{\max}(s)} $$
    $$ \tilde{g}_s(\mathbf{x}) = (1 - \eta^{-1}) \cdot |\mathbf{x} - s| + \eta^{-1} \cdot \iota(s) $$
    $$ g(\mathbf{x}) = \min_{s \in S} \tilde{g}_s(\mathbf{x}) $$
    where $\kappa_{\max}(s)$ is the maximum absolute curvature at $s$ [S1].
3.  **Cull Guidance Field:** Remove samples dominated by others to reduce memory and query time (see [Minimal Guidance Field Culling](#minimal-guidance-field-culling)) [S1].
4.  **Seed the Front:** Compute seed points (e.g., using efficient algorithms like [6] in [S1]) and project them onto the surface using gradient descent. Initialize the `Active` front set with these seeds [S1].
5.  **Advance Front:** While `Active` is not empty:
    a. Get a front from `Active`.
    b. Check if it is OK to add a triangle using parameters $\rho$ and $\eta$.
    c. If OK, add the triangle to the front.
    d. If not OK, get the interfering front.
    e. If the interfering front is the same as the current front, split the front into two new fronts ($f_1, f_2$), remove the old front, and add the two new fronts to `Active`.
    f. If the interfering front is different, merge the current front and the interfering front into a new front, remove both from `Active`, and add the new front to `Active` [S1].
6.  **Create New Triangles:** Determine the correct edge size from the guidance field $g$. Project the new point onto the surface using the dihedral angle of the edge as a free parameter and searching for the isovalue [S1].

**Curvature Calculation:**
The maximum curvature $\kappa_{\max}$ is derived from the geometry tensor $G$. Let $\mathbf{n} = \nabla f / |\nabla f|$ be the normalized gradient. The projection matrix onto the tangent plane is:
$$ P = I - \mathbf{n}\mathbf{n}^T $$
The bound for $\kappa_{\max}$ is given by:
$$ \kappa_{\max} \le \|P\| \cdot \|H\| \cdot \|P\| \cdot \|I/|\nabla f|\| $$
where $\|\cdot\|$ denotes the Frobenius norm $\|M\|_2 = \sqrt{\sum m_{ij}^2}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def TRIANGULATE(f, grad_f, hess_f, k, rho, eta):
    # 1. Generate Samples
    # Create jittered random points in cells neighboring the surface
    S = []
    for cell in neighboring_cells(f, k):
        points = jittered_random_points_in_cell(cell)
        for p in points:
            # Project onto surface via gradient descent
            projected = gradient_descent_projection(f, k, p)
            if projected is not None:
                S.append(projected)
    
    # 2. Compute Guidance Field
    # For each sample, compute ideal edge length and cone function
    guidance_field_samples = []
    for s in S:
        n = normalize(grad_f(s))
        H = hess_f(s)
        P = identity_matrix() - outer_product(n, n)
        # Compute kappa_max bound
        norm_P = frobenius_norm(P)
        norm_H = frobenius_norm(H)
        norm_grad_inv = 1.0 / abs(dot(grad_f(s), grad_f(s))) ** 0.5
        kappa_max_bound = norm_P * norm_H * norm_P * norm_grad_inv
        
        # Ideal edge length
        iota_s = (2 * sin(rho / 2)) / kappa_max_bound
        
        # Cone function parameters: slope (1 - eta^-1), intercept eta^-1 * iota_s
        guidance_field_samples.append({
            'point': s,
            'iota': iota_s,
            'slope': 1.0 - 1.0/eta,
            'intercept': (1.0/eta) * iota_s
        })
    
    # 3. Cull Guidance Field
    guidance_field_samples = CULL(guidance_field_samples, rho, eta)
    
    # 4. Seed Fronts
    Active = SEED_FRONTS(S) # Uses efficient seed point algorithm [6]
    
    # 5. Advance Front
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

def OK_TO_ADD_TRIANGLE(front, rho, eta):
    # Check front interference using guidance field properties
    # Returns True if adding the triangle does not cause interference
    # and respects curvature bounds
    pass

def GET_INTERFERING_FRONT(front):
    # Find another front that interferes with the current front
    # based on the guidance field g(x)
    pass

def ADD_TRIANGLE_TO_FRONT(front, rho, eta):
    # Determine correct edge size from guidance field g
    # Project new point onto surface using dihedral angle
    # Add triangle to mesh and update front
    pass
```

## Subtopics

### Minimal Guidance Field Culling
A technique to reduce the number of samples in a guidance field by removing samples that are 'dominated' by others [S1]. A sample $s_1$ dominates $s_2$ if the edge length required for curvature at $s_1$ is always smaller than that of $s_2$, ensuring the guidance field remains sufficient and Lipschitz-continuous [S1].
**Steps:**
1. Insert all points in the guidance field into a 4-dimensional kd-tree. The coordinates for each sample $s$ are $[s_x, s_y, s_z, s_r]$, where $[s_x, s_y, s_z]$ is the sample's location in $\mathbb{R}^3$, and $s_r$ is the height of the function $\tilde{g}_s$ at $s$ [S1].
2. Sort the samples by their maximum curvature ($\kappa_{max}$) in descending order [S1].
3. Iterate through the sorted samples. For each sample $s_i$, if it is not already marked as dominated, construct a cone $c$ defined by $s_i$, user parameters $\rho$ and $\eta$, apex $[s_x, s_y, s_z, s_r]$, axis $[0, 0, 0, 1]$, and angle $\tan^{-1}(\eta/(\eta-1))$ [S1].
4. Perform a kd-tree query to find all points lying inside the cone $c$. Mark all such points as dominated. If all children of a kd-tree node are marked, mark the node itself to prune branches [S1].
5. Discard all marked samples from the guidance field $g(\mathbf{x})$ [S1].

## Operating characteristics
**Time Complexity:** The guidance field evaluation is efficient using kd-tree queries, avoiding the need to splat values into a regular grid [S1]. Culling typically removes a large percentage of sample points, resulting in significantly faster guidance field queries and lower memory usage [S1]. Table 1 in [S1] shows that culling can reduce running time from ~27 minutes to ~7 minutes for a large dataset, despite the overhead of culling.
**Memory Complexity:** Reduced significantly by culling unnecessary samples. In some cases, less than half of a percent of points remain in the guidance field after culling [S1].
**Strengths:** Robust front interference tests proven by guidance field properties; prevents sharp changes in triangle size; bounds Hausdorff distance and aspect ratio; no need for regular grid splatting [S1].
**Limitations:** Assumes the surface is smooth; requires sufficient sampling to guarantee feature reproduction; computing curvature bounds can be expensive for high-gradient regions [S1].
**When to use:** When high-quality, robust isosurface extraction is required without downstream processing, and the surface is smooth.
**When not to use:** For surfaces with sharp features that violate the smoothness assumption, or when extremely high performance is needed and some mesh quality degradation is acceptable (consider parallel variants or marching cubes).

## Background
The method is based on the advancing front algorithm of Schreiner et al. [45] in [S1], which introduced a solid theoretical foundation for advancing front algorithms using a guidance field. It addresses limitations of previous methods that relied on heuristics for front interference and suffered from poorly shaped triangles due to sharp curvature changes [S1]. The guidance field construction ensures that $g(\mathbf{x})$ is bounded above by $2 \sin(\rho)/\kappa_{\max}$ and has a gradient magnitude never greater than $1-\eta^{-1}$ [S1].

## Sources
[S1] High-Quality Extraction of Isosurfaces from Regular and Irregular Grids, https://doi.org/10.1109/tvcg.2006.149, fetched 2026-10-26
