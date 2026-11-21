# Gaussian Graph Network


## Overview
Gaussian Graph Network (GGN) is a feed-forward method that generates efficient and generalizable 3D Gaussian representations from multi-view images by modeling relations between Gaussian groups from different views using a Gaussian Graph structure, enabling message passing and aggregation at the Gaussian level [S1]. Unlike previous methods that simply combine pixel-aligned Gaussians from multiple views without modeling relations, GGN explicitly models these relations via Gaussian Graphs, allowing each Gaussian to benefit from its connected Gaussian groups through feature fusion [S1]. This approach mitigates artifacts and extra memory costs associated with naive combination, enabling the model to benefit from increased input views rather than suffering performance degradation [S1].

## How it works
**Inputs:** A set of $N$ input images $I = \{I_i\} \in \mathbb{R}^{N \times H \times W \times 3}$ and their corresponding camera parameters $C = \{c_i\}$ [S1].
**Outputs:** Gaussian parameters $(\mu, \Sigma, \alpha, c)$ representing the 3D scene [S1].

**Data Structures:**
*   **Node $v_i$:** Represents a group of pixel-aligned Gaussians from view $i$. It consists of a mean position $\mu_i \in \mathbb{R}^{HW \times 3}$ and a feature vector $f_i \in \mathbb{R}^{HW \times D}$ [S1].
*   **Edge Weight Matrix $E_{s \to k}$:** Of size $HW \times HW$, containing binary values (0 or 1) indicating pixel correspondence between Gaussians in group $s$ and group $k$ [S1]. Specifically, $e_{ij}^{s \to k} = 1$ if $\psi_{proj}(\mu_{s,i}, c_s) = \psi_{proj}(\mu_{k,j}, c_s)$, and 0 otherwise, where $\mu_{s,i}$ is the center of the $i$-th Gaussian in group $v_s$ and $\psi_{proj}$ projects the mean to the image plane of camera $c_s$ [S1].
*   **Adjacency Matrix $A$:** Of size $N \times N$, where $a_{ij} = 1$ if $i=j$, and $a_{ij} = \psi_{overlap}(v_i, v_j)$ if $i \neq j$, with $\psi_{overlap}$ computing the overlap ratio between views [S1].
*   **Degree Matrix $D$:** Diagonal matrix where $d_{ii} = \sum_j a_{ij}$ [S1].

**Parameters:**
*   $N$: Number of input images [S1].
*   $h$: Number of graph layers in the Gaussian Graph Network [S1].
*   $n$: Number of top edges to preserve during graph pruning [S1].
*   $W$: Learnable weight matrix of size $C \times D$ used in the linear operation [S1].
*   $C$: Feature dimension from the 2D backbone [S1].
*   $D$: Output feature dimension for the graph layers [S1].

**Steps:**

1.  **Feature Extraction:** Extract image features $F$ from input images $I$ using a 2D backbone $\Phi_{image}$:
    $$F = \Phi_{image}(I)$$
    where $F = \{F_i\} \in \mathbb{R}^{N \times \frac{H}{4} \times \frac{W}{4} \times C}$ [S1].

2.  **Initial Gaussian Prediction:** Predict mean positions $\mu_i$ and features $f_i$ for pixel-aligned Gaussians for each view $i$:
    $$\mu_i = \psi_{unproj}(\Phi_{depth}(F_i), c_i) \in \mathbb{R}^{HW \times 3}$$
    $$f_i = \Phi_{feat}(F_i) \in \mathbb{R}^{HW \times D}$$
    where $\Phi_{depth}$ and $\Phi_{feat}$ are neural networks predicting depth maps and Gaussian features, and $\psi_{unproj}$ is the unprojection operation [S1].

3.  **Graph Construction:** Construct a Gaussian Graph $G$ with nodes $V = \{v_i\} = \{(\mu_i, f_i)\}$ for $i=1$ to $N$ [S1]. Define the adjacency matrix $A$ based on view overlap [S1].

4.  **Graph Pruning:** Prune the graph by preserving edges with the top $n$ weights and ignoring others to limit computational complexity [S1].

5.  **Normalization:** Compute the degree matrix $D$ where $d_{ii} = \sum_j a_{ij}$, and calculate the scaled adjacency matrix $\tilde{A}$:
    $$\tilde{A} = D^{-1}A \quad \text{or} \quad \tilde{A} = D^{-1/2}AD^{-1/2}$$
    [S1].

6.  **Message Passing:** For each graph layer $k$ from 1 to $h$, update features $f_i$ for each node $i=1$ to $N$ using the Gaussian-level linear operation:
    $$f_i \leftarrow \sigma\left( \sum_{j=1}^N \tilde{a}_{ij} E_{j \to i} f_j W \right)$$
    where $\sigma$ is a non-linearity, $E_{j \to i}$ is the edge weight matrix between node $j$ and node $i$, and $W$ is the learnable weight matrix [S1].

7.  **Gaussian Pooling:** Apply Gaussian pooling to aggregate Gaussians from connected subgraphs $G_s$:
    $$(\mu, f) \leftarrow \sum_{s=1}^l \phi_{pooling}(G_s)$$
    If a subgraph $G_s$ has one node $v_s$, $\phi_{pooling}(G_s) = v_s$. Otherwise, aggregation is performed over the connected components [S1].

8.  **Parameter Prediction:** Predict remaining Gaussian parameters from the pooled features $f$:
    $$R, S, \alpha, c \leftarrow \psi_R(f), \psi_S(f), \psi_\alpha(f), \psi_c(f)$$
    Compute covariance $\Sigma = RSS^T R^T$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_graph_network(I, C, h, n):
    """
    I: List of N images, shape (H, W, 3)
    C: List of N camera parameters
    h: Number of graph layers
    n: Number of top edges to preserve
    """
    # Step 1: Feature Extraction
    # Phi_image is a 2D backbone
    F = phi_image(I) 
    # F is a list of N tensors, each shape (H/4, W/4, C)

    # Step 2: Initial Gaussian Prediction
    mu_list = []
    f_list = []
    for i in range(len(I)):
        # Phi_depth predicts depth, psi_unproj unprojects to 3D
        mu_i = psi_unproj(phi_depth(F[i]), C[i]) 
        # mu_i shape: (H*W, 3)
        
        # Phi_feat predicts features
        f_i = phi_feat(F[i])
        # f_i shape: (H*W, D)
        
        mu_list.append(mu_i)
        f_list.append(f_i)

    # Step 3 & 4: Graph Construction and Pruning
    # Compute overlap ratios for adjacency matrix A
    A = compute_adjacency_matrix(mu_list, C)
    # A is (N, N)
    
    # Prune graph: keep top n edges per node or globally? 
    # Source says "preserving edges with the top n weights". 
    # Assuming global top n or per-node top n. 
    # Implementation detail: Sort edges by weight, keep top n.
    A = prune_graph(A, n)

    # Step 5: Normalization
    D = compute_degree_matrix(A)
    # Option 1: Symmetric normalization
    # D_inv_sqrt = matrix_inverse_sqrt(D)
    # A_tilde = D_inv_sqrt @ A @ D_inv_sqrt
    # Option 2: Row normalization
    # D_inv = matrix_inverse(D)
    # A_tilde = D_inv @ A
    
    # Step 6: Message Passing
    for k in range(h):
        new_f_list = []
        for i in range(len(mu_list)):
            sum_val = zeros_like(f_list[i])
            for j in range(len(mu_list)):
                if A[i, j] > 0:
                    # Compute edge weight matrix E_{j->i}
                    # E_{j->i} is (HW, HW) binary matrix
                    E_ji = compute_edge_matrix(mu_list[j], mu_list[i], C[j])
                    
                    # Linear operation: E_{j->i} @ f_j @ W
                    # f_j is (HW, D), W is (D, D_out)
                    # E_{j->i} @ f_j results in (HW, D)
                    # Then @ W results in (HW, D_out)
                    term = E_ji @ f_list[j] @ W
                    sum_val += A_tilde[i, j] * term
            
            # Apply non-linearity sigma
            f_new = sigma(sum_val)
            new_f_list.append(f_new)
        f_list = new_f_list

    # Step 7: Gaussian Pooling
    # Identify connected subgraphs G_s
    subgraphs = find_connected_components(A)
    
    pooled_mu = []
    pooled_f = []
    for s in range(len(subgraphs)):
        G_s = subgraphs[s]
        # If G_s has one node, return it
        if len(G_s) == 1:
            idx = G_s[0]
            pooled_mu.append(mu_list[idx])
            pooled_f.append(f_list[idx])
        else:
            # Aggregate Gaussians from connected subgraph
            # Source implies summation or specific pooling function
            # Assuming simple summation for demonstration if not specified
            sub_mu = sum(mu_list[idx] for idx in G_s)
            sub_f = sum(f_list[idx] for idx in G_s)
            pooled_mu.append(sub_mu)
            pooled_f.append(sub_f)

    # Step 8: Parameter Prediction
    # Combine pooled features
    final_f = concat(pooled_f, axis=0)
    
    R = psi_R(final_f)
    S = psi_S(final_f)
    alpha = psi_alpha(final_f)
    c = psi_c(final_f)
    
    # Compute covariance
    # Sigma = R @ S @ S^T @ R^T
    # Note: Source says Sigma = RSS^T R^T. 
    # Assuming R is rotation matrix (3x3) and S is scale matrix (3x3)
    # Or R is rotation quaternion/axis-angle and S is scale vector.
    # Standard 3DGS: Sigma = R S S^T R^T
    Sigma = R @ (S @ S.T) @ R.T
    
    return mu, Sigma, alpha, c

def compute_edge_matrix(mu_s, mu_k, c):
    """
    mu_s: (HW, 3) means for group s
    mu_k: (HW, 3) means for group k
    c: camera parameters for projection
    Returns: (HW, HW) binary matrix
    """
    HW = mu_s.shape[0]
    E = zeros((HW, HW))
    
    # Project means to image plane
    proj_s = psi_proj(mu_s, c) # (HW, 2) pixel coordinates
    proj_k = psi_proj(mu_k, c) # (HW, 2) pixel coordinates
    
    for i in range(HW):
        for j in range(HW):
            # Check if pixel coordinates match
            if proj_s[i] == proj_k[j]:
                E[i, j] = 1
            else:
                E[i, j] = 0
    return E

def compute_adjacency_matrix(mu_list, C):
    """
    Computes overlap ratio between views.
    """
    N = len(mu_list)
    A = zeros((N, N))
    for i in range(N):
        for j in range(N):
            if i == j:
                A[i, j] = 1
            else:
                A[i, j] = psi_overlap(mu_list[i], mu_list[j], C[i], C[j])
    return A

def compute_degree_matrix(A):
    D = zeros_like(A)
    for i in range(A.shape[0]):
        D[i, i] = sum(A[i, :])
    return D

def matrix_inverse_sqrt(M):
    # Compute inverse square root of diagonal matrix M
    # M is diagonal, so just invert sqrt of diagonal elements
    inv_sqrt_diag = 1.0 / sqrt(diag(M))
    return diag(inv_sqrt_diag)
```

## Operating characteristics
*   **Time Complexity:** The complexity is dominated by the graph construction and message passing. The edge weight matrix computation is $O(N^2 \cdot HW^2)$ if done naively, but pruning reduces the number of active edges. Message passing involves matrix multiplications of size $HW \times HW$ scaled by the number of edges.
*   **Memory Complexity:** Stores $N$ Gaussian groups, each with $HW$ Gaussians. The adjacency matrix is $N \times N$, and edge weight matrices are $HW \times HW$. Pruning reduces memory usage for edges.
*   **Performance:** Outperforms previous methods under different input settings with higher rendering quality and fewer Gaussian representations [S1]. Performance improves as the number of input views increases, unlike previous methods which degrade [S1].
*   **Strengths:** Efficient and generalizable representations; models relations between Gaussians; benefits from more input views [S1].
*   **Limitations:** Computational complexity of graph operations; requires careful pruning to limit complexity [S1].

## Background
Gaussian Graph Network (GGN) was proposed to address the limitations of methods that simply combine pixel-aligned Gaussians from multiple views without modeling relations, which leads to artifacts and extra memory cost [S1]. The method was introduced in the paper "Gaussian Graph Network: Learning Efficient and Generalizable Gaussian Representations from Multi-view Images" [S1]. It builds on the concept of pixelSplat [4] and MVSplat [6] but extends them by introducing Gaussian Graphs for message passing and aggregation [S1]. Experiments were conducted on large-scale RealEstate10K and ACID datasets [S1].

## Sources
[S1] Gaussian Graph Network: Learning Efficient and Generalizable Gaussian Representations from Multi-view Images, https://doi.org/10.48550/arXiv.2503.16338, fetched 2026-10-29
