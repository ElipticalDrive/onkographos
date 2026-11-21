# Strivec

Also called: Sparse Tri-Vector Radiance Fields

## Overview
Strivec is a neural scene representation that models a 3D radiance field using sparsely distributed, compactly factorized local tensor feature grids [S1]. It solves the inefficiency of global tensor representations by exploiting scene sparsity directly, avoiding the modeling of empty space, while maintaining local smoothness and coherence through tensor factorization [S1]. Unlike its parent [TensoRF](3026-tensorf.md), which uses a global tensor with vector-matrix decomposition, Strivec employs a cloud of local tensors decomposed via Canonical Polyadic (CP) decomposition into triple vectors, enabling high-quality reconstruction with significantly fewer parameters [S1].

## How it works
**Inputs:** A set of calibrated images; coarse scene geometry (pre-acquired via fast RGB$\sigma$ volume reconstruction like DVGO or multi-view stereo like Point-NeRF) [S1].
**Outputs:** A continuous radiance field defined by volume density $\sigma$ and view-dependent color $c$ at any 3D location [S1].

**Data Structures:**
1.  **Tensor Cloud ($\mathcal{T}$):** A set of $N$ local tri-vector tensors $\mathcal{T} = \{\tau_n \mid n = 1, \dots, N\}$ [S1].
2.  **Local Tensor ($\tau$):** Each tensor $\tau$ is associated with a center position $\mathbf{p}$ and covers a local cuboid space $\omega$ with edge length $l$ [S1].
3.  **Tensor Components:** Each local tensor $\tau$ is factorized into $R$ rank-1 components using CP decomposition [S1].
    *   **Density Tensor ($\mathcal{G}^\sigma$):** A 3D tensor of shape $I \times J \times K$ representing the density grid [S1].
    *   **Appearance Tensor ($\mathcal{G}^c$):** A 4D tensor of shape $I \times J \times K \times P$ representing the appearance grid, where $P$ is the number of appearance channels (lower dimension than spatial modes) [S1].
4.  **Tri-Vector Factors:** For a tensor of dimension $M$, the CP decomposition yields vectors $\mathbf{v}_r^0, \dots, \mathbf{v}_r^M$ for each rank $r$ [S1].

**Parameters:**
*   $N$: Total number of local tensors.
*   $M$: Number of nearest tensors selected to cover an arbitrary location $\chi$ [S1].
*   $R$: Number of rank-1 components (CP rank) in the decomposition [S1].
*   $l$: Edge length of the local cuboid space $\omega$ [S1].
*   $\psi$: A small Multi-Layer Perceptron (MLP) used to regress color [S1].

**Algorithm:**

1.  **Pre-acquisition:** Obtain coarse scene geometry using a fast method (e.g., DVGO or Point-NeRF) [S1].
2.  **Distribution:** Distribute local tensors $\tau_n$ around the actual scene surface based on the coarse geometry, defining the radiance field domain $\Omega = \bigcup_{n=1}^N \omega_n$ [S1].
3.  **Query:** For a query point $\chi \in \Omega$:
    a. Identify the set of $M$ nearest tensors $\{\tau_{n_1}, \dots, \tau_{n_M}\}$ that cover or are nearest to $\chi$ [S1].
    b. For each selected tensor $\tau$, extract density features $\mathcal{G}^\sigma(\chi)$ and appearance features $\mathcal{G}^c(\chi)$ using the CP-decomposed tri-vectors [S1].
    c. Aggregate the extracted features from the $M$ tensors [S1].
4.  **Regression:**
    a. Compute volume density $\sigma_\chi$ by aggregating density features: $\sigma_\chi = A_\sigma(\{\mathcal{G}^\sigma(\chi)\})$ [S1].
    b. Compute view-dependent color $c_\chi$ by passing the aggregated density features and viewing direction $\mathbf{d}$ through the MLP $\psi$: $c_\chi = \psi(A_\sigma(\{\mathcal{G}^\sigma(\chi)\}), \mathbf{d})$ [S1].

**CP Decomposition Formula:**
Each local tensor $\tau \in \mathbb{R}^{I_1 \times I_2 \times \dots \times I_M}$ is factorized as [S1]:
$$ \tau = \sum_{r=1}^R \lambda_r \mathbf{v}_r^0 \otimes \mathbf{v}_r^1 \otimes \dots \otimes \mathbf{v}_r^M $$
where $\otimes$ denotes the outer product, and the weighting factor $\lambda_r$ is absorbed into the vectors $\mathbf{v}_r^k$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def strivec_query(chi, tensor_cloud, M, psi_mlp):
    """
    Query the Strivec radiance field at location chi.
    
    Args:
        chi: 3D coordinate (x, y, z)
        tensor_cloud: List of N local tensors. Each tensor is a dict with keys:
            'p': center position (3,)
            'l': edge length (float)
            'v_sigma': Density tri-vectors, shape (R, I, J, K) -> actually stored as 
                       list of 3 vectors of shape (R, dim) or similar. 
                       Assuming CP factors: v0 (I, R), v1 (J, R), v2 (K, R) for density.
            'v_color': Appearance tri-vectors, shape (R, I, J, K, P). 
                       Stored as v0 (I, R), v1 (J, R), v2 (K, R), v3 (P, R).
            'R': CP rank
        M: Number of nearest tensors to select
        psi_mlp: MLP function taking aggregated features and direction
        
    Returns:
        sigma: Volume density
        c: View-dependent color
    """
    # Step 1: Select M nearest tensors
    # Calculate distance from chi to center p of each tensor
    # Note: The source says "select M nearest tensors that cover chi". 
    # If chi is not covered by any, nearest neighbors are used.
    distances = []
    for n, tau in enumerate(tensor_cloud):
        p = tau['p']
        dist = np.linalg.norm(chi - p)
        distances.append((dist, n))
    
    distances.sort(key=lambda x: x[0])
    selected_indices = [idx for _, idx in distances[:M]]
    selected_tensors = [tensor_cloud[i] for i in selected_indices]
    
    # Step 2: Extract and Aggregate Features
    aggregated_density_features = []
    aggregated_appearance_features = []
    
    for tau in selected_tensors:
        # Extract density feature at chi using CP decomposition
        # tau['v_sigma'] contains factors for density tensor G^sigma
        # G^sigma is 3D: I x J x K
        # Factors: v0 (I, R), v1 (J, R), v2 (K, R)
        v0, v1, v2 = tau['v_sigma'] # Shapes: (I, R), (J, R), (K, R)
        
        # Interpolate/evaluate CP tensor at chi relative to tensor bounds
        # This requires mapping chi to local coordinates within the cuboid omega
        # and interpolating along the vectors.
        # For simplicity, assuming a helper function 'cp_evaluate' exists
        # which handles the coordinate mapping and outer product summation.
        sigma_feat = cp_evaluate_density(chi, tau, v0, v1, v2)
        aggregated_density_features.append(sigma_feat)
        
        # Extract appearance feature at chi using CP decomposition
        # tau['v_color'] contains factors for appearance tensor G^c
        # G^c is 4D: I x J x K x P
        # Factors: v0 (I, R), v1 (J, R), v2 (K, R), v3 (P, R)
        v0_c, v1_c, v2_c, v3_c = tau['v_color']
        color_feat = cp_evaluate_color(chi, tau, v0_c, v1_c, v2_c, v3_c)
        aggregated_appearance_features.append(color_feat)
        
    # Step 3: Aggregate
    # A_sigma aggregates density features. Typically a sum or mean.
    # The source implies aggregation across the M tensors.
    agg_sigma = np.mean(aggregated_density_features, axis=0) # Example aggregation
    
    # Step 4: Regress Color
    # c = psi(agg_sigma, d)
    # Note: d (viewing direction) is not passed in this function signature 
    # but is required for the MLP.
    # We assume d is available in the context or passed separately.
    # Here we return the aggregated features for the caller to pass to psi.
    return agg_sigma, aggregated_appearance_features # Simplified return

def cp_evaluate_density(chi, tau, v0, v1, v2):
    """
    Evaluate the density tensor at chi using CP factors.
    """
    # Map global chi to local coordinates within the cuboid defined by tau['p'] and tau['l']
    # This involves normalization to [0,1] or [-1,1] range of the tensor dimensions
    local_coords = map_to_local(chi, tau['p'], tau['l'])
    
    # Interpolate each vector at the corresponding local coordinate
    # v0 corresponds to dimension 0 (x), v1 to 1 (y), v2 to 2 (z)
    val0 = interpolate_vector(v0, local_coords[0])
    val1 = interpolate_vector(v1, local_coords[1])
    val2 = interpolate_vector(v2, local_coords[2])
    
    # Sum over R components: sum_r (val0_r * val1_r * val2_r)
    # Assuming vectors are shape (R, dim) or (dim, R). 
    # Standard CP: tau = sum_r v_r^0 o v_r^1 o v_r^2
    # If v0 is (I, R), then v0[:, r] is the r-th vector for dim 0.
    # We need to interpolate the r-th vector at the specific coordinate.
    
    R = v0.shape[1]
    density_val = 0.0
    for r in range(R):
        # Interpolate the r-th component of each vector
        # Note: Implementation of interpolate_vector depends on storage format
        # Assuming vectors are stored as (dim, R)
        comp0 = interpolate_vector(v0[:, r], local_coords[0])
        comp1 = interpolate_vector(v1[:, r], local_coords[1])
        comp2 = interpolate_vector(v2[:, r], local_coords[2])
        density_val += comp0 * comp1 * comp2
        
    return density_val

def map_to_local(chi, p, l):
    """
    Map global coordinate chi to local [0, 1] coordinates within the cuboid.
    """
    # Assuming the cuboid is axis-aligned and centered at p with edge length l
    # Local coord = (chi - (p - l/2)) / l
    return (chi - (p - l / 2.0)) / l

def interpolate_vector(vector, coord):
    """
    Interpolate a 1D vector at a normalized coordinate [0, 1].
    Uses linear interpolation.
    """
    # vector shape: (size,)
    # coord in [0, 1]
    # Map to indices
    size = len(vector)
    idx = coord * (size - 1)
    i = int(np.floor(idx))
    j = int(np.ceil(idx))
    i = np.clip(i, 0, size - 1)
    j = np.clip(j, 0, size - 1)
    w = idx - i
    return (1 - w) * vector[i] + w * vector[j]
```

**Edge Cases and Pitfalls:**
*   **Empty Space:** If $\chi$ is not covered by any tensor, the nearest neighbor selection must handle distances gracefully. The source notes that tensors are distributed around the actual scene surface, so empty space is largely avoided [S1].
*   **Precision:** CP decomposition involves sums of outer products. Numerical stability should be considered when $R$ is large, though Strivec uses compact factors [S1].
*   **Boundary Conditions:** Interpolation at the boundaries of the local cuboid $\omega$ must be handled correctly to avoid discontinuities between adjacent tensors [S1].

## Subtopics

### Multi-scale Tensor Grids
Strivec uses multi-scale tensor grids to discover geometry and appearance commonalities and exploit spatial coherence at multiple local scales [S1]. This allows the representation to capture both fine details and coarse structures efficiently.

### Comparison with TensoRF
TensoRF uses a global tensor and vector-matrix decomposition, while Strivec uses a cloud of local tensors and CP decomposition (triple vectors). Local tensors are less complex and lower rank, reducing the required number of CP components per tensor [S1].

### Comparison with Global Representations
Unlike global representations (TensoRF, Instant-NGP), Strivec exploits scene sparsity directly by distributing tensors only where geometry exists [S1].

### Comparison with Local Representations
Unlike local representations (Plenoxels, Point-NeRF), Strivec uses local smoothness and coherence via tensor factorization, correlating features inside a local box rather than optimizing them independently [S1].

## Operating characteristics
*   **Time Complexity:** Query time depends on $M$ (number of nearest tensors) and $R$ (CP rank). For each query, $M$ tensors are evaluated, each requiring $O(R \cdot \text{dim})$ operations for interpolation and summation.
*   **Memory Complexity:** Memory usage is proportional to $N \cdot R \cdot (\text{spatial dims} + \text{appearance dims})$. Since $N$ is sparse and $R$ is small due to local compactness, memory usage is significantly lower than global tensor methods [S1].
*   **Performance:** Strivec achieves state-of-the-art rendering quality on synthetic and real datasets, outperforming TensoRF and Instant-NGP while using significantly fewer model parameters [S1].
*   **Strengths:** Efficient scene modeling, compact representation, exploitation of sparsity and local coherence [S1].
*   **Limitations:** Requires pre-acquisition of coarse geometry, which adds an initial processing step [S1].

## Background
Strivec was proposed by Quankai Gao, Qiangeng Xu, Hao Su, Ulrich Neumann, and Zexiang Xu from USC, UC San Diego, and Adobe Research [S1]. It was published in ICCV 2023 [S1]. The method builds on the tensor decomposition principles of TensoRF [S1] but adapts them to a sparse, local setting. It draws inspiration from both global tensor representations (TensoRF, Instant-NGP) and local representations (Plenoxels, Point-NeRF), aiming to combine the best of both worlds [S1].

## Sources
[S1] Strivec: Sparse Tri-Vector Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01611, fetched 2026-10-26
