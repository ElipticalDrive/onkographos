# Vector-Matrix-Matrix (VMM) Factorization

Also called: VMM decomposition

## Overview
Vector-Matrix-Matrix (VMM) factorization is a compact representation technique for light field tensors that models radiance and geometry by decomposing feature probes into shared core factors and probe-specific basis factors [S1]. It distinguishes itself from standard CP decomposition by introducing a hybrid structure where a Vector-Matrix (VM) core captures multilinear interactions shared across local probes, while a Matrix (M) basis factor handles individual probe details, enabling efficient spatial sharing and multi-scale pyramid storage [S1].

## How it works

### Inputs and Outputs
*   **Input:** A set of $L$ discrete spherical probes $\mathcal{T} = \{\tau_l \in \mathbb{R}^{H \times W \times D}\}_{l=1}^L$, where each probe $\tau_l$ represents a $D$-layered spherical grid located at position $p_l$ [S1].
*   **Output:** A continuous feature field $\tau_l(\mathbf{x})$ for any 3D coordinate $\mathbf{x}$, reconstructed via weighted blending of neighboring probes [S1].

### Data Structures and Layout
The decomposition utilizes three types of factor tensors. The layout and typical resolutions for optimization are [S1]:

1.  **Core Vector ($\ddot{V}_{c,r}$):**
    *   **Shape:** $\mathbb{R}^D$
    *   **Role:** Shared among all local probes; captures depth interactions.
    *   **Resolution:** Starts at 64, upscaled to 1024 during optimization.
2.  **Core Matrix ($\ddot{M}_{c,r}$):**
    *   **Shape:** $\mathbb{R}^{H \times W}$
    *   **Role:** Shared among all local probes; captures angular interactions.
    *   **Resolution:** Starts at $(64, 128)$, upscaled to $(128, 256)$ during optimization.
3.  **Basis Matrix ($M_{l,r}$):**
    *   **Shape:** $\mathbb{R}^{H \times W}$
    *   **Role:** Specific to probe $l$; designated exclusively for each individual probe.
    *   **Resolution:** Fixed at $(256, 512)$ during optimization.

**Parameters:**
*   $R$: Number of factor components (rank). Set to 32 [S1].
*   $F$: Number of basis factor components reduced via a linear layer before aggregation. Set to 2 [S1].
*   $D$: Dimension of the core vector (depth of spherical grid) [S1].
*   $H, W$: Spherical dimensions of the matrix factors, providing full angular coverage [S1].

### Coordinate Transformation
To query values on the low-dimensional factors, the input 3D coordinate $\mathbf{x} = (x, y, z)$ is transformed into local normalized depth and spherical coordinates relative to a probe center $p$ [S1]:

$$
\tilde{t} = \|\mathbf{x} - p\|_2
$$

$$
t = \frac{1}{\tilde{t} + 1}
$$

$$
\tilde{\mathbf{v}} = \left( \arccos \left( \frac{z}{\tilde{t}} \right), \arctan \left( \frac{y}{x} \right) \right)
$$

Here, $t$ represents the normalized depth, and $\tilde{\mathbf{v}}$ represents the spherical angles [S1].

### Formulas
The discrete tensor element $\tau_l^{ijk}$ for probe $l$ at indices $i, j, k$ is factorized as a sum of $R$ scalar products using the Hadamard product ($\circ$) [S1]:

$$
\tau_l^{ijk} = \sum_{r=1}^R \ddot{V}_{c,r}^k \circ \ddot{M}_{c,r}^{ij} \circ M_{l,r}^{ij}
$$

Where:
*   $\ddot{V}_{c,r} \in \mathbb{R}^D$ is the core vector component $r$ for core index $c$.
*   $\ddot{M}_{c,r} \in \mathbb{R}^{H \times W}$ is the core matrix component $r$ for core index $c$.
*   $M_{l,r} \in \mathbb{R}^{H \times W}$ is the basis matrix component $r$ for probe $l$.
*   $c, l$ are core and basis factor indices respectively.
*   $i, j, k$ denote the three modes (angular and depth) [S1].

For continuous field reconstruction, linear interpolation $(\cdot)$ is applied to each vector and matrix when querying a target position $\mathbf{x}$ [S1]:

$$
\tau_l(\mathbf{x}) = \sum_{r=1}^R \ddot{V}_{c,r}(\mathbf{x}) \circ \ddot{M}_{c,r}(\mathbf{x}) \circ M_{l,r}(\mathbf{x})
$$

To simplify computation, scalar values are concatenated into an $R$-channel vector: $\ddot{V}_c(\tilde{t}_c) = [\ddot{V}_{c,1}(\tilde{t}_c), \dots, \ddot{V}_{c,R}(\tilde{t}_c)]$. Similarly for $\ddot{M}_c$ and $M_l$. The combined feature for probe $l$ at $\mathbf{x}$ is denoted as $G_l(\mathbf{x}) = \ddot{V}_c(\tilde{t}_c) \circ \ddot{M}_c(\tilde{v}_c) \circ M_l(v_l)$ [S1]. The final tensor value is recovered via summation: $\tau_l(\mathbf{x}) = \sum G_l(\mathbf{x})$, though a neural decoder may be included instead [S1].

### Algorithm Steps

1.  **Neighbor Selection:** For a target camera center, select $K$ neighboring feature probes based on the distance between the target camera center and the probe centers. Use $\hat{L}=16$ neighboring basis factors for representation [S1].
2.  **Coordinate Transformation:** For each selected probe $l$ and target point $\mathbf{x}$, compute local coordinates $t$ and $\tilde{\mathbf{v}}$ using the transformation equations above [S1].
3.  **Factor Interpolation:** Perform linear interpolation on the core vector $\ddot{V}_{c,r}$, core matrix $\ddot{M}_{c,r}$, and basis matrix $M_{l,r}$ using the local coordinates to obtain interpolated values $\ddot{V}_{c,r}(\mathbf{x})$, $\ddot{M}_{c,r}(\mathbf{x})$, and $M_{l,r}(\mathbf{x})$ [S1].
4.  **Weighted Blending:** Apply efficient weighted blending on each factor group (core and basis) across the $K$ selected probes. This is done *before* performing the final matrix multiplication to avoid multiplying factors for each probe individually [S1].
5.  **Factor Multiplication:** Compute the Hadamard product of the blended core vector, blended core matrix, and blended basis matrix for each rank $r$ [S1].
6.  **Summation:** Sum the results over all $R$ ranks to obtain the feature tensor $\tau_l(\mathbf{x})$ [S1].
7.  **Decoding (Optional):** Pass the aggregated features through a neural network $\Phi$ to predict density $\alpha_i$ and blended radiance weights $w_{i,l}$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def vmm_factorization_query(probes, core_vectors, core_matrices, basis_matrices, 
                            target_point, probe_centers, weights):
    """
    Reconstructs the feature tensor at a target point using VMM factorization.
    
    Args:
        probes: List of probe indices selected (size K).
        core_vectors: Tensor of shape (C, R, D) - Core vectors for C cores.
        core_matrices: Tensor of shape (C, R, H, W) - Core matrices for C cores.
        basis_matrices: Tensor of shape (L, R, H, W) - Basis matrices for L probes.
        target_point: 3D coordinate (x, y, z).
        probe_centers: Array of probe center coordinates (L, 3).
        weights: Blending weights for the K selected probes (K,).
        
    Returns:
        feature_tensor: Reconstructed feature tensor (R,).
    """
    K = len(probes)
    R = core_vectors.shape[1]
    D = core_vectors.shape[2]
    H, W = core_matrices.shape[2], core_matrices.shape[3]
    
    # Initialize blended factors
    # Blended core vector: (R, D)
    blended_core_vec = np.zeros((R, D))
    # Blended core matrix: (R, H, W)
    blended_core_mat = np.zeros((R, H, W))
    # Blended basis matrix: (R, H, W)
    blended_basis_mat = np.zeros((R, H, W))
    
    for k in range(K):
        probe_idx = probes[k]
        probe_center = probe_centers[probe_idx]
        w = weights[k]
        
        # 1. Coordinate Transformation
        dist_sq = np.sum((target_point - probe_center) ** 2)
        t_tilde = np.sqrt(dist_sq)
        t = 1.0 / (t_tilde + 1.0)
        
        # Spherical coordinates
        # Handle edge case where t_tilde is 0 to avoid division by zero
        if t_tilde < 1e-8:
            theta = 0.0
            phi = 0.0
        else:
            theta = np.arccos(target_point[2] / t_tilde)
            phi = np.arctan2(target_point[1], target_point[0])
            
        # 2. Linear Interpolation of Factors
        # Note: In practice, this involves trilinear/bilinear interpolation on the grids.
        # Here we assume helper functions for interpolation.
        
        # Interpolate Core Vector (D dimension)
        # Core vector index c is typically fixed or selected based on scene scale.
        # Assuming c=0 for simplicity as per typical single-core usage in examples.
        c = 0
        interp_core_vec = interpolate_linear(core_vectors[c], t, D)
        
        # Interpolate Core Matrix (H, W dimensions)
        interp_core_mat = interpolate_bilinear(core_matrices[c], theta, phi, H, W)
        
        # Interpolate Basis Matrix (H, W dimensions)
        interp_basis_mat = interpolate_bilinear(basis_matrices[probe_idx], theta, phi, H, W)
        
        # 3. Weighted Blending
        blended_core_vec += w * interp_core_vec
        blended_core_mat += w * interp_core_mat
        blended_basis_mat += w * interp_basis_mat
        
    # 4. Factor Multiplication (Hadamard Product)
    # Result shape: (R, H, W) -> then sum over H,W to get (R,) or keep as tensor
    # The formula sums over R. The output is typically a feature vector per rank.
    # However, the formula shows tau_l(x) = sum_r (V o M o M).
    # This results in a tensor of shape (H, W) if we sum over R? 
    # No, the formula is element-wise in i,j,k. 
    # Let's follow the vector concatenation description:
    # G_l(x) = V_c(x) o M_c(x) o M_l(x) -> This is element-wise product of vectors/matrices.
    # If V is (R, D), M_c is (R, H, W), M_l is (R, H, W).
    # The product is (R, H, W).
    # Then tau_l(x) = sum_r G_l(x)? No, sum_r is over the rank dimension.
    # The formula: tau_l^{ijk} = sum_r V^k o M^{ij} o M^{ij}.
    # This implies the output is a scalar per (i,j,k) if we sum over r?
    # Actually, V is (R, D), M_c is (R, H, W), M_l is (R, H, W).
    # The Hadamard product is element-wise.
    # So for each rank r, we have a matrix (H,W) and a vector (D).
    # The result is a tensor (R, H, W, D)? No.
    # Let's look at Eq 5: tau_l^{ijk} = sum_r V_{c,r}^k o M_{c,r}^{ij} o M_{l,r}^{ij}.
    # i,j are spatial/angular, k is depth.
    # V_{c,r}^k is a scalar (k-th element of vector r).
    # M_{c,r}^{ij} is a scalar.
    # M_{l,r}^{ij} is a scalar.
    # So for each r, we compute a scalar product.
    # Then we sum over r.
    # The result is a tensor of shape (H, W, D).
    
    # Re-evaluating based on "concatenate all scalar values into an R-channel vector"
    # This suggests the final output might be kept as R channels before summation or decoding.
    # But the formula explicitly sums over R.
    
    # Implementation of the sum over R:
    # We have blended_core_vec (R, D), blended_core_mat (R, H, W), blended_basis_mat (R, H, W).
    # We need to compute sum_r (core_vec_r * core_mat_r * basis_mat_r).
    
    # Expand dimensions for broadcasting
    # core_vec: (R, 1, 1, D)
    # core_mat: (R, H, W, 1)
    # basis_mat: (R, H, W, 1)
    
    # This is complex to broadcast directly to (H, W, D).
    # Instead, iterate or use einsum.
    
    # Using einsum:
    # V: (R, D)
    # Mc: (R, H, W)
    # Mb: (R, H, W)
    # Result: sum_r (V_{r,k} * Mc_{r,i,j} * Mb_{r,i,j}) for each i,j,k
    
    # This results in a tensor of shape (H, W, D).
    
    # However, the text says "G_l(x) = V_c o M_c o M_l ... recover tau_l(x) using a simple sum operation".
    # If G_l is (R, H, W) [assuming D is handled separately or V is part of the matrix? No, V is vector].
    # Let's stick to the formula:
    
    # tau_l(x) is a tensor of shape (H, W, D).
    
    # Efficient computation:
    # 1. Element-wise multiply core_mat and basis_mat: (R, H, W)
    # 2. Multiply by core_vec: (R, H, W, D) -> broadcast V over H,W
    # 3. Sum over R: (H, W, D)
    
    product = blended_core_mat * blended_basis_mat  # (R, H, W)
    # Expand V to (R, 1, 1, D)
    product_expanded = product[:, :, :, np.newaxis] * blended_core_vec[:, np.newaxis, np.newaxis, :] # (R, H, W, D)
    feature_tensor = np.sum(product_expanded, axis=0) # (H, W, D)
    
    return feature_tensor

def interpolate_linear(tensor, t, dim):
    """Placeholder for linear interpolation along dimension dim."""
    # In practice, this maps t to indices and interpolates.
    # For VMM, t is normalized depth.
    return tensor[int(t * (tensor.shape[0]-1))] # Simplified for pseudocode

def interpolate_bilinear(tensor, theta, phi, H, W):
    """Placeholder for bilinear interpolation on spherical grid."""
    # Map theta, phi to i, j indices
    i = int((theta / np.pi) * (H - 1))
    j = int(((phi + np.pi) / (2 * np.pi)) * (W - 1))
    return tensor[i, j] # Simplified
```

### Edge Cases and Pitfalls
*   **Division by Zero:** When the target point coincides with a probe center, $\tilde{t} = 0$. The formula for $t$ involves division by $\tilde{t}+1$, which is safe, but the spherical coordinate calculation $\arccos(z/\tilde{t})$ requires handling $\tilde{t}=0$ explicitly to avoid NaNs [S1].
*   **Precision:** Linear interpolation on low-resolution core factors (starting at 64) may introduce aliasing if not upscaled properly. The method uses progressive upscaling to mitigate this [S1].
*   **Boundary Conditions:** The spherical coordinates $\theta$ and $\phi$ must be correctly mapped to the matrix indices $H$ and $W$ to ensure full angular coverage [S1].

## Operating characteristics

*   **Time Complexity:** The factorization reduces the complexity of storing and querying light field tensors. By sharing core factors, the number of parameters is significantly reduced compared to storing full tensors for each probe. The weighted blending before multiplication reduces the number of matrix multiplications from $K \times R$ to $1 \times R$ (after blending) [S1].
*   **Memory Complexity:** The memory usage is dominated by the core factors (shared) and the basis factors (per probe). With $R=32$, $D=1024$, $H=128$, $W=256$ for cores and $H=256$, $W=512$ for basis, the memory footprint is compact relative to the scene size [S1].
*   **Strengths:**
    *   **Compactness:** Achieves compact encoding while preserving locality [S1].
    *   **Efficiency:** Fast reconstruction and aliasing-free rendering via soft localization and blending [S1].
    *   **Multi-scale:** Stores 3D points in a multi-scale pyramid (mipmaps) format, modeling nearby areas at higher resolution and distant areas at lower resolution [S1].
*   **Limitations:**
    *   **Resolution Dependency:** Performance depends on the resolution of the core and basis factors. Lower resolutions may lose detail [S1].
    *   **Probe Selection:** Requires selecting $K$ neighboring probes, which adds computational overhead during query time [S1].
*   **When to Use:** Suitable for novel view synthesis in scenes with arbitrary scales where compact representation and efficient rendering are prioritized [S1].

## Background

*   **Origin:** The Vector-Matrix-Matrix (VMM) factorization was introduced in the paper "NeLF-Pro: Neural Light Field Probes for Multi-Scale Novel View Synthesis" [S1].
*   **Development:** It was developed to improve grid-based representations for handling scenes with arbitrary scales within a compact and efficient framework. It builds on the concept of light field feature probes and introduces a novel factorization technique to uncover hidden relationships and patterns within the scene [S1].
*   **Usage:** It is used in novel view synthesis tasks, particularly for large-scale scenes like Google Earth, KITTI360, and City Walk, as demonstrated in the evaluation section of the source paper [S1].

## Sources

[S1] NeLF-Pro: Neural Light Field Probes for Multi-Scale Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.01875, fetched 2026-10-27
