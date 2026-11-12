# TetraSDF

Also called: Analytic Isosurface Extraction, Analytic Isosurface Extraction with Multi-resolution Tetrahedral Grid, TetraSDF extraction

## Overview
TetraSDF is an analytic isosurface extraction framework for neural Signed Distance Functions (SDFs) that represents the SDF as a ReLU Multi-Layer Perceptron (MLP) composed with a multi-resolution tetrahedral positional encoder [S1]. It solves the problem of discretization error inherent in sampling-based methods like Marching Cubes, which deviate from the true zero-level set, and the lack of high-frequency detail capture in plain ReLU MLPs with analytic marching [S2, unofficial]. It sets itself apart by preserving a global Continuous Piecewise Affine (CPWA) structure via barycentric interpolation, enabling exact zero-level set extraction without grid resolution artifacts [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A trained neural SDF model $f(\mathbf{x}) = \nu^{(M)}(\tau(\mathbf{x}))$, where $\nu^{(M)}$ is an $M$-layer ReLU network and $\tau$ is the multi-resolution tetrahedral positional encoder. The model parameters (weights, biases, hash table features) and hyperparameters ($L, N_{min}, N_{max}$) are required.
*   **Output:** A triangle mesh representing the exact zero-level set $\{\mathbf{x} | f(\mathbf{x}) = 0\}$.

**Data Structures:**
1.  **Multi-resolution Tetrahedral Grid:** Defined by $L$ levels. Level $\ell$ has resolution $N_\ell$. Each unit cube cell at level $\ell$ is decomposed into 6 congruent tetrahedra [S1].
2.  **Barycentric Mask $\mathbf{m}(\mathbf{x})$:** A binary vector of length $4L$. For level $\ell$, $m_i^{(\ell)}(\mathbf{x}) = 1$ if barycentric weight $w_i^{(\ell)}(\mathbf{x}) > \epsilon_b$, else $0$. Zero entries indicate the point lies on a face (1 zero), edge (2 zeros), or vertex (3 zeros) of the tetrahedron [S1].
3.  **Region Indicator $\mathbf{r}(\mathbf{x})$:** A vector of length $4L$. For level $\ell$, $\mathbf{r}^{(\ell)}(\mathbf{x}) = [\mathbf{a}^{(\ell)}(\mathbf{x}), t^{(\ell)}(\mathbf{x})]$, where $\mathbf{a}^{(\ell)}(\mathbf{x}) = \lfloor N_\ell \mathbf{x} \rfloor$ is the grid anchor (lexicographically smallest vertex of the containing cube) and $t^{(\ell)}(\mathbf{x}) \in \{0, \dots, 5\}$ is the tetrahedron offset within that cube [S1].
4.  **Sign Vector $\mathbf{s}_k^{(m)}(\mathbf{x})$:** Tracks the state of neurons in the ReLU network. Entry $\gamma_j^{(i)}(\mathbf{x})$ is $+1$ if preactivation $> \epsilon_s$, $-1$ if $< -\epsilon_s$, and $0$ if $|\text{preactivation}| \le \epsilon_s$ [S1].

**Parameters:**
*   $L$: Number of resolution levels.
*   $N_{min}, N_{max}$: Minimum and maximum grid resolutions.
*   $\epsilon_s$: Small positive constant for decision boundary detection.
*   $\epsilon_b$: Small threshold for barycentric mask.
*   $\mathbf{A}^*$: Input preconditioner matrix.

**Algorithm Steps:**

1.  **Input Preconditioning:**
    Compute the Jacobian $\mathbf{J}_{\mathcal{T}}$ of the barycentric weights. Calculate the average local metric $\mathbf{M} = 6 \sum_{\mathcal{T}} \mathbf{J}_{\mathcal{T}}^T \mathbf{J}_{\mathcal{T}}$. Apply the global linear preconditioner $\mathbf{A}^*$ to the input coordinates: $\mathbf{x}' = \mathbf{A}^* \mathbf{x}$. This isotropizes the metric, reducing the condition number from 16.39 to 5.05 [S1].

2.  **Initial Skeleton Extraction:**
    Represent the encoder-induced polyhedral cells $\mathcal{C}_{\mathbf{x}}$ as a union of parallel-plane sets. Group shared normals $\mathcal{N} = \{\mathbf{n}_0, \dots\}$ with level-dependent offsets $d^{(i)}$.
    *   Extract vertices as intersections of 3+ planes with linearly independent normals.
    *   Extract edges as intersections of 2+ non-parallel planes.
    This forms the initial vertex set $V$ and edge set $E$ [S1].

3.  **Edge Subdivision (Joint Tracking):**
    Process all ReLU decision boundaries $H_k^{(m)}$ in increasing order of layer $m$ and neuron index $k$. For each boundary:
    *   Iterate through all edges $(\mathbf{x}_0, \mathbf{x}_1)$ in $E$.
    *   Check for sign change: if $\gamma_k^{(m)}(\mathbf{x}_0) \gamma_k^{(m)}(\mathbf{x}_1) < 0$, the edge crosses the boundary.
    *   Compute intersection weight: $w = d_0 / (d_0 - d_1)$, where $d_0 = \nu_k^{(m)}(\mathbf{x}_0)$ and $d_1 = \nu_k^{(m)}(\mathbf{x}_1)$ [S1].
    *   Compute intersection point: $\hat{\mathbf{x}}_{0,1} = (1 - w)\mathbf{x}_0 + w\mathbf{x}_1$ [S1].
    *   Add $\hat{\mathbf{x}}_{0,1}$ to $V$. Split edge $(\mathbf{x}_0, \mathbf{x}_1)$ into $(\mathbf{x}_0, \hat{\mathbf{x}}_{0,1})$ and $(\hat{\mathbf{x}}_{0,1}, \mathbf{x}_1)$ in $E$ [S1].
    *   *Note:* To handle folded hyperplanes consistently, use sign vectors to determine valid candidate edges formed by intersection points. Two intersection points are connected if their sign vectors match on all non-zero entries, and if they share at least two zero entries (indicating they lie on the same geometric feature) [S1].

4.  **Mesh Extraction:**
    From the final sets $V$ and $E$, extract faces. This typically involves identifying polygons formed by the intersection of the zero-level set with the polyhedral cells, using the connectivity established in $E$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def compute_grid_resolution(N_min, N_max, L):
    """Compute geometric progression ratio and level resolutions."""
    gamma = np.exp((np.log(N_max) - np.log(N_min)) / (L - 1))
    N_l = np.floor(N_min * gamma ** np.arange(L)).astype(int)
    return N_l

def barycentric_weights(pos_frac, tet_idx):
    """
    Compute barycentric weights for a point within a specific tetrahedron.
    pos_frac: (N, 3) fractional coordinates in [0, 1].
    tet_idx: (N,) integer indices of tetrahedra (0-5).
    Returns: (N, 4) barycentric weights.
    """
    # Define the 6 tetrahedra partitioning a unit cube.
    # Vertices of the cube are indexed 0..7.
    # Standard decomposition (e.g., from [S1] Fig 3):
    # T0: [0, 1, 5, 7], T1: [0, 3, 5, 7], T2: [0, 2, 5, 7],
    # T3: [0, 4, 5, 7], T4: [0, 6, 5, 7], T5: [0, 7, 5, 7] 
    # Note: Exact vertex indices depend on the specific convention. 
    # Using a generic lookup for A and B matrices derived from tet geometry.
    
    # Precomputed matrices A (6x4x3) and B (6x4) for the 6 tetrahedra
    # such that weights = A @ pos_frac + B
    # These matrices map fractional cube coordinates to barycentric coords of the tet.
    # Implementation detail: These are constant for the specific subdivision scheme.
    A = torch.tensor([...], dtype=torch.float32) # Shape (6, 4, 3)
    B = torch.tensor([...], dtype=torch.float32) # Shape (6, 4)
    
    A_sel = A[tet_idx] # (N, 4, 3)
    B_sel = B[tet_idx] # (N, 4)
    
    weights = torch.bmm(A_sel, pos_frac.unsqueeze(-1)).squeeze(-1) + B_sel
    return weights

def get_tetrahedron_index(pos_frac):
    """
    Determine which of the 6 tetrahedra contains the point based on sorted fractional coords.
    """
    # Sort fractional coordinates descending to get permutation
    perm = torch.argsort(pos_frac, dim=-1, descending=True, stable=True)
    # Encode permutation into a key
    enc = perm[:, 0] * 9 + perm[:, 1] * 3 + perm[:, 2]
    # Map key to tetrahedron index using a lookup table
    # PERM_TO_TETRA_IDX is a tensor of size 27 (3^3) mapping enc to tet_idx
    key_lut = torch.tensor([...], dtype=torch.long) # Size 27
    tet_idx = key_lut[enc]
    return tet_idx

def extract_mesh(model, points, grid_resolution, L, N_min, N_max, epsilon_s, epsilon_b):
    """
    Simplified extraction logic for a batch of points.
    Note: Full extraction requires iterative edge subdivision over the entire domain,
    not just point-wise evaluation. This pseudocode illustrates the core evaluation
    and sign-checking logic used within the subdivision steps.
    """
    # 1. Preconditioning (if enabled)
    # x_precond = A_star @ points
    
    # 2. Grid Positioning
    pos_scaled = points * (grid_resolution - 1)
    pos_grid_float = torch.floor(pos_scaled)
    pos_frac = pos_scaled - pos_grid_float
    
    # 3. Identify Cell
    cube_indices = pos_grid_float.long()
    tet_idx = get_tetrahedron_index(pos_frac)
    
    # 4. Barycentric Weights
    weights = barycentric_weights(pos_frac, tet_idx)
    
    # 5. Barycentric Mask
    m = (weights > epsilon_b).float() # (N, 4)
    
    # 6. Region Indicator
    # a = floor(N_l * x) -> derived from pos_grid_float scaled by level ratio
    # t = tet_idx
    # r = concat(a, t) across levels
    
    # 7. Network Evaluation for Sign Vectors
    # Encode point using multi-resolution encoder
    features = []
    for l in range(L):
        # Scale point for level l
        N_l = compute_grid_resolution(N_min, N_max, L)[l]
        # ... hash grid lookup and barycentric interpolation ...
        # h_i = hash_table[level_l][index]
        # feat_l = sum(w_v * h_v)
        features.append(feat_l)
    
    x_encoded = torch.cat(features, dim=-1)
    
    # Pass through ReLU MLP
    # nu(x) = rho(M) o sigma(M-1) o ... o rho(1)
    x_net = x_encoded
    for i in range(model.M):
        x_net = model.linear_layers[i](x_net)
        if i < model.M - 1:
            x_net = torch.relu(x_net)
            
    # 8. Sign Vector Calculation
    # For each neuron, determine +1, -1, or 0
    # This is complex for the full network; typically done layer by layer
    # during the subdivision process to track regions.
    
    return weights, tet_idx, cube_indices
```

## Subtopics

### [Tensorized Edge Construction from Planes](13262-tensorized-edge-construction-from-planes.md)
The geometric core of the TetraSDF framework, responsible for extracting the zero-level set from a continuous piecewise affine (CPWA) scalar field defined on a tetrahedral grid. It is distinguished from marching cubes or standard marching tetrahedra by operating directly on the analytic plane intersections rather than sampling discrete voxels, enabling exact zero-level set extraction without grid resolution artifacts.

## Operating characteristics
*   **Time Complexity:** Depends on the number of ReLU decision boundaries and the complexity of the polyhedral complex. The edge subdivision algorithm processes boundaries sequentially, which can be expensive for high-capacity networks with many linear regions.
*   **Memory Complexity:** Scales with the number of vertices and edges in the extracted mesh, as well as the hash table size for the encoder. Model capacity 'large' uses ~8 GB GPU memory, 'medium' ~1.5 GB [S2, unofficial].
*   **Performance:** Matches or surpasses existing grid-based encoders in reconstruction accuracy. On the Stanford dataset, the fraction of faces with minimum angle below 5 degrees is 11.83% (vs 10.05% for AM and 10.94% for TropicalNeRF) [S1].
*   **Strengths:** Exact zero-level set extraction; preserves fine geometric detail; no discretization error.
*   **Limitations:** Requires a CPWA representation (ReLU MLP + specific encoder); complex implementation due to joint tracking of encoder cells and ReLU regions; potential for skinny triangles (though post-processing can mitigate this at the cost of exactness) [S1].
*   **When to use:** When exact agreement with the learned neural SDF is critical and the model uses a compatible CPWA architecture.
*   **When not to use:** For non-CPWA networks (e.g., those using trilinear interpolation or non-piecewise affine activations) where sampling-based methods are the only option.

## Background
TetraSDF was introduced by Seonghun Oh, Youngjung Uh, and Jin-Hwa Kim from Yonsei University, NAVER AI Lab, and Seoul National University [S1]. The paper "TetraSDF: Analytic Isosurface Extraction with Multi-resolution Tetrahedral Grid" was published on arXiv in 2025 [S1]. It addresses the limitations of Marching Cubes (discretization error) and plain ReLU MLPs (lack of high-frequency detail) by combining the expressiveness of grid-based encoders with the exactness of analytic extraction. The method builds on the principles of tetrahedral representation and traversal [4396-tetrahedral-representation-and-traversal.md].

## Sources
[S1] TetraSDF: Analytic Isosurface Extraction with Multi-resolution Tetrahedral Grid, https://arxiv.org/abs/2511.16273v4, fetched 2026-10-26
[S2] naver-ai/TetraSDF, https://github.com/naver-ai/TetraSDF, fetched 2026-10-27 (unofficial)
