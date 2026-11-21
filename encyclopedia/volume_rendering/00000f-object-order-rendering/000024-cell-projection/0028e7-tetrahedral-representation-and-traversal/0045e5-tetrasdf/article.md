# TetraSDF

Also called: Analytic Isosurface Extraction, Analytic Isosurface Extraction with Multi-resolution Tetrahedral Grid, TetraSDF extraction

## Overview
TetraSDF is an analytic isosurface extraction framework for neural Signed Distance Functions (SDFs) that represents the SDF as a ReLU Multi-Layer Perceptron (MLP) composed with a multi-resolution tetrahedral positional encoder [S1]. It solves the problem of discretization error inherent in sampling-based methods like Marching Cubes by preserving a global Continuous Piecewise Affine (CPWA) structure, enabling exact extraction of the network's zero-level set without geometric deviation [S1][S2, unofficial]. Unlike plain ReLU MLPs with analytic marching which fail to capture fine high-frequency details, TetraSDF retains the expressiveness of grid-based encoders while allowing exact mesh extraction [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A trained neural SDF model $f = \nu^{(M)} \circ \tau$, where $\nu^{(M)}$ is an $M$-layer ReLU network and $\tau$ is the multi-resolution tetrahedral positional encoder. The encoder parameters include the number of levels $L$, minimum resolution $N_{\min}$, maximum resolution $N_{\max}$, and hash table size.
*   **Output:** A triangular mesh representing the zero-level set $\{x \mid f(x) = 0\}$.

**Data Structures:**
1.  **Multi-resolution Tetrahedral Grid:** Defined by $L$ levels. Level $\ell$ has resolution $N_\ell$. Each unit cube cell at level $\ell$ is decomposed into 6 congruent tetrahedra [S1].
2.  **Region Indicator $\mathbf{r}(\mathbf{x})$:** A vector of size $4L$. For each level $\ell$, it concatenates the grid anchor $\mathbf{a}^{(\ell)}(\mathbf{x}) = \lfloor N_\ell \mathbf{x} \rfloor$ (lexicographically smallest vertex of the containing cube) and the tetrahedron offset $t^{(\ell)}(\mathbf{x}) \in \{0, \dots, 5\}$ [S1].
3.  **Barycentric Mask $\mathbf{m}(\mathbf{x})$:** A binary vector of size $4L$. For each level $\ell$, entry $i$ is $1$ if barycentric weight $w_i^{(\ell)}(\mathbf{x}) > \epsilon_b$, else $0$. Zeros indicate boundaries (faces, edges, or vertices) [S1].
4.  **Sign Vector $\mathbf{s}_k^{(m)}(\mathbf{x})$:** Tracks the state of the $k$-th neuron in the $m$-th layer. Entry $\gamma_j^{(i)}(\mathbf{x})$ is $+1$ if preactivation $> \epsilon_s$, $-1$ if $< -\epsilon_s$, and $0$ if $|\text{preactivation}| \le \epsilon_s$ [S1].
5.  **Skeleton $(V, E)$:** Sets of vertices and edges representing the polyhedral complex and linear regions.

**Parameters:**
*   $L$: Number of resolution levels [S1].
*   $N_{\min}, N_{\max}$: Min/Max grid resolutions [S1].
*   $\epsilon_s$: Small positive constant for decision boundary detection [S1].
*   $\epsilon_b$: Small threshold for barycentric mask [S1].
*   `grid_resolution`: Integer resolution for scaling points, typically 256 [S2, unofficial].
*   `use_A`: Boolean flag for input preconditioner, enabled by default [S2, unofficial].

**Algorithm Steps:**

1.  **Input Preconditioning (Optional but recommended):**
    Compute the Jacobian $\mathbf{J}_{\mathcal{T}}$ of the barycentric weights. Calculate the average local metric $\mathbf{M} = \frac{1}{6} \sum_{\mathcal{T}} \mathbf{J}_{\mathcal{T}}^\top \mathbf{J}_{\mathcal{T}}$. Apply a global linear preconditioner $\mathbf{A}^*$ such that $\mathbf{x}' = \mathbf{A}^* \mathbf{x}$ isotropizes $\mathbf{M}$, reducing the condition number from 16.39 to 5.05 [S1]. If `use_A` is false, $\mathbf{x}' = \mathbf{x}$ [S2, unofficial].

2.  **Initial Skeleton Extraction:**
    Represent the encoder-induced polyhedral cells $\mathcal{C}_{\mathbf{x}}$ as unions of parallel-plane sets. Group shared normals $\mathcal{N} = \{\mathbf{n}_0, \dots\}$ with level-dependent offsets $d^{(i)}$. Extract vertices as intersections of 3+ planes with linearly independent normals. Extract edges as intersections of 2+ non-parallel planes. This forms the initial vertex set $V$ and edge set $E$ [S1].

3.  **Edge Subdivision (Joint Tracking):**
    Process all decision boundaries $H_k^{(m)}$ of the ReLU network in increasing order of layer $m$ and neuron index $k$ [S1].
    For each boundary $H_k^{(m)}$:
    a. Iterate through all edges $(\mathbf{x}_0, \mathbf{x}_1) \in E$.
    b. Compute sign vector entries $\gamma_k^{(m)}(\mathbf{x}_0)$ and $\gamma_k^{(m)}(\mathbf{x}_1)$.
    c. If $\gamma_k^{(m)}(\mathbf{x}_0) \gamma_k^{(m)}(\mathbf{x}_1) < 0$, the edge crosses the boundary.
    d. Compute preactivation values $d_0 = \nu_k^{(m)}(\mathbf{x}_0)$ and $d_1 = \nu_k^{(m)}(\mathbf{x}_1)$.
    e. Calculate intersection weight $w = \frac{d_0}{d_0 - d_1}$.
    f. Compute intersection point $\hat{\mathbf{x}}_{0,1} = (1 - w)\mathbf{x}_0 + w\mathbf{x}_1$.
    g. Add $\hat{\mathbf{x}}_{0,1}$ to $V$.
    h. Split edge $(\mathbf{x}_0, \mathbf{x}_1)$ into $(\mathbf{x}_0, \hat{\mathbf{x}}_{0,1})$ and $(\hat{\mathbf{x}}_{0,1}, \mathbf{x}_1)$ in $E$ [S1].

4.  **Grid-Aware Subdivision (Handling Polyhedral Cells):**
    The edge subdivision above handles ReLU boundaries. To handle the encoder's polyhedral cells, use the region indicator $\mathbf{r}(\mathbf{x})$ and barycentric mask $\mathbf{m}(\mathbf{x})$. When a point lies on a boundary (zero in $\mathbf{m}$), identify neighboring cells by perturbing $\mathbf{r}(\mathbf{x})$ according to the zero pattern. Precompute lookup tables for neighbor configurations (anchor displacements) for each level [S1].

5.  **Face Extraction:**
    From the final sets $V$ and $E$, determine valid faces. A candidate edge formed by two intersection points is valid if:
    a. Their sign vectors match on all non-zero entries (zeros are wildcards).
    b. If a vertex pair shares at least two zero entries in their masks, they are connected [S1].
    Construct triangles from these connected vertices.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def extract_mesh(model, points, grid_resolution=256, use_A=True):
    """
    Extracts mesh from TetraSDF model.
    model: Contains encoder tau and MLP nu.
    points: Tensor of shape (N, 3) representing the bounding box grid points.
    """
    # 1. Preconditioning
    if use_A:
        # Apply preconditioner A* derived from encoder metric
        # A* isotropizes the average local metric M
        points = apply_preconditioner(points)
    
    # 2. Initial Skeleton Extraction
    # Extract vertices and edges from plane intersections of the multi-resolution grid
    V, E = extract_initial_skeleton(model.encoder)
    
    # 3. Edge Subdivision for ReLU Boundaries
    # Iterate through layers and neurons
    for m in range(model.num_layers):
        for k in range(model.num_neurons[m]):
            # Get boundary H_k^(m)
            # For each edge in E, check if it crosses the boundary
            new_edges = []
            for (x0, x1) in E:
                # Compute preactivations
                d0 = model.nu.preactivation(x0, m, k)
                d1 = model.nu.preactivation(x1, m, k)
                
                # Check sign change
                # Use epsilon_s for robustness
                if (d0 > model.epsilon_s and d1 < -model.epsilon_s) or \
                   (d0 < -model.epsilon_s and d1 > model.epsilon_s):
                    # Compute intersection
                    w = d0 / (d0 - d1)
                    x_hat = (1 - w) * x0 + w * x1
                    
                    # Add vertex and split edge
                    V.append(x_hat)
                    new_edges.append((x0, x_hat))
                    new_edges.append((x_hat, x1))
                else:
                    new_edges.append((x0, x1))
            E = new_edges

    # 4. Grid-Aware Subdivision (Polyhedral Cells)
    # This step ensures the mesh respects the encoder's cell boundaries
    # Using region indicators and barycentric masks to find neighbors
    # and subdivide edges crossing cell boundaries
    E = subdivide_for_polyhedral_cells(E, model.encoder)
    
    # 5. Face Extraction
    # Connect vertices based on sign vector matching and mask overlap
    faces = extract_faces(V, E)
    
    return V, faces

def extract_initial_skeleton(encoder):
    """
    Extracts initial V and E from the multi-resolution tetrahedral grid.
    Groups shared normals and offsets to find plane intersections.
    """
    V = []
    E = []
    # Implementation details from Appendix A.4:
    # 1. Identify unique normals N from the 6 tetrahedra subdivision convention.
    # 2. For each normal, collect offsets d^(i) across all levels L.
    # 3. Vertices: Intersections of 3+ planes with linearly independent normals.
    # 4. Edges: Intersections of 2+ non-parallel planes.
    # This is implemented using parallel tensor operations.
    return V, E

def subdivide_for_polyhedral_cells(E, encoder):
    """
    Subdivides edges that cross polyhedral cell boundaries defined by the encoder.
    """
    new_edges = []
    for (x0, x1) in E:
        # Check if edge crosses any cell boundary
        # Use barycentric masks m(x) and region indicators r(x)
        m0 = encoder.get_barycentric_mask(x0)
        m1 = encoder.get_barycentric_mask(x1)
        r0 = encoder.get_region_indicator(x0)
        r1 = encoder.get_region_indicator(x1)
        
        # If masks differ, the edge crosses a cell boundary
        if not torch.equal(m0, m1):
            # Find intersection point with the boundary plane
            # The boundary is defined by w_i^(l)(x) = epsilon_b
            # Solve for intersection along the edge
            x_hat = find_boundary_intersection(x0, x1, encoder)
            new_edges.append((x0, x_hat))
            new_edges.append((x_hat, x1))
        else:
            new_edges.append((x0, x1))
    return new_edges

def find_boundary_intersection(x0, x1, encoder):
    """
    Finds the point where the edge (x0, x1) crosses a barycentric weight boundary.
    """
    # Linear interpolation of barycentric weights along the edge
    # w(x) = J_T * x + b_T
    # Solve w_i^(l)(x) = epsilon_b for each level l and vertex i
    # Return the closest intersection point within the segment
    pass # Implementation relies on tensorized Jacobian-vector products

def extract_faces(V, E):
    """
    Constructs faces from V and E based on sign vector matching.
    """
    faces = []
    # For each vertex, find neighbors in E
    # Group edges by sign vector compatibility
    # If two edges share a vertex and their sign vectors match on non-zero entries,
    # and they share at least two zero entries in masks, they form a face corner.
    # Construct triangles from these compatible edge pairs.
    return faces
```

## Subtopics

### [Tensorized Edge Construction from Planes](13262-tensorized-edge-construction-from-planes.md)
Computes intersection lines between pairs of planes sharing a common vertex in a tetrahedral mesh to extract the zero-level set of a continuous piecewise affine scalar field. It distinguishes itself from voxel-based methods by operating directly on analytic plane intersections, enabling exact zero-level set extraction without grid resolution artifacts.

## Operating characteristics
*   **Time Complexity:** Depends on the number of ReLU neurons and the complexity of the polyhedral complex. The edge subdivision algorithm processes boundaries sequentially, which can be expensive for deep networks with many neurons. However, the tensorized scheme for handling polyhedral structures enables efficient GPU implementation [S1].
*   **Memory Complexity:** Scales with the number of resolution levels $L$ and the hash table size. Model capacity 'large' uses ~8 GB GPU memory, 'medium' ~1.5 GB, 'small' less [S2, unofficial].
*   **Performance:** Matches or surpasses existing grid-based encoders in reconstruction accuracy [S1]. On the Stanford dataset, the fraction of faces with minimum angle below 5 degrees is 11.83% for TetraSDF, compared to 10.05% for AM and 10.94% for TropicalNeRF [S1].
*   **Strengths:** Exact agreement with the learned zero-level set; preserves fine geometric detail without extraction error; handles high-frequency SDFs via multi-resolution encoder.
*   **Limitations:** Triangle regularity (minimum angle) is not optimized during extraction, potentially resulting in skinny triangles (11.83% < 5 degrees on Stanford dataset) [S1]. Requires custom CUDA extensions for efficient training and extraction [S2, unofficial].
*   **When to use:** When exact geometric fidelity to the neural SDF is required, and fine detail preservation is critical.
*   **When not to use:** When computational efficiency of extraction is paramount and some discretization error is acceptable (use Marching Cubes); when triangle regularity is strictly required without post-processing.

## Background
TetraSDF was introduced by Seonghun Oh, Youngjung Uh, and Jin-Hwa Kim from Yonsei University, NAVER AI Lab, and Seoul National University [S1]. The paper "TetraSDF: Analytic Isosurface Extraction with Multi-resolution Tetrahedral Grid" was published on arXiv in November 2025 [S1]. It addresses the limitations of Marching Cubes (discretization error) and plain ReLU MLPs (lack of high-frequency detail) by combining multi-resolution tetrahedral positional encoding with analytic isosurface extraction [S1]. The method extends the edge subdivision algorithm to operate jointly on encoder-induced polyhedral cells and ReLU MLP linear regions [S1].

## Sources
[S1] TetraSDF: Analytic Isosurface Extraction with Multi-resolution Tetrahedral Grid, https://arxiv.org/abs/2511.16273v4, fetched 2026-10-26
[S2] naver-ai/TetraSDF, https://github.com/naver-ai/TetraSDF, fetched 2026-10-27 (unofficial)
