# SVD-based Cloud Volume Compression

Also called: SVD Compression for Cloud Volumes

## Overview
This method compresses a database of extracted cloud volumes by projecting their voxel density vectors onto a low-dimensional subspace defined by the left singular vectors of the dataset matrix [S1]. It differs from its parent, [PCA-based TSDF Compression](12932-pca-based-tsdf-compression.md), by operating directly on the raw voxel density values of normalized cloud blocks rather than on Signed Distance Field (SDF) or TSDF representations [S1]. While PCA relies on the covariance matrix of the data, this method explicitly constructs the data matrix and performs Singular Value Decomposition (SVD) to determine the basis, allowing for a direct threshold-based truncation of the basis set [S1].

## How it works
**Inputs:** A set of $N$ extracted cloud volumes $D_n$ (where $n = 1, \dots, N$) [S1].
**Outputs:** A compressed database containing the truncated basis vectors $\{u_k\}_{k=1}^K$ and their corresponding coefficients $\{w_{nk}\}_{n=1, k=1}^{N, K}$ [S1].

**Data Structures:**
1.  **Normalized Volume ($S_n$):** A cubic grid of size $m \times m \times m$ [S1].
2.  **Density Vector ($x_n$):** A column vector of size $M \times 1$, where $M = m^3$ [S1]. It contains the voxel densities $\rho(p)$ of $S_n$ flattened in a consistent order [S1].
3.  **Data Matrix ($A$):** An $M \times N$ matrix formed by concatenating the density vectors [S1].
4.  **Basis Vectors ($U$):** The left singular vectors of $A$. Specifically, the first $K$ columns of $U$, denoted $u_1, \dots, u_K$, serve as the basis. Each $u_k$ is a vector of size $M \times 1$ [S1].
5.  **Coefficients ($W$):** A matrix of size $N \times K$ where entry $(n, k)$ is $w_{nk}$ [S1].

**Numbered Steps:**
1.  **Normalization:** For each cloud volume $D_n$, extract the bounding box of the connected cloud region. Resize/interpolate this volume to a fixed cubic resolution of $m \times m \times m$ to create $S_n$ [S1].
2.  **Vectorization:** Flatten each $S_n$ into the column vector $x_n$ of size $M = m^3$ [S1].
3.  **Matrix Construction:** Assemble the data matrix $A = [x_1 \dots x_N]$ of size $M \times N$ [S1].
4.  **SVD Decomposition:** Compute the Singular Value Decomposition of $A$ [S1].
5.  **Basis Selection:** Determine the smallest integer $K$ such that the sum of the first $K$ singular values satisfies the threshold condition [S1].
6.  **Coefficient Calculation:** For each cloud volume $n$, compute the projection coefficients onto the selected basis vectors using orthogonality [S1].
7.  **Storage:** Store the basis vectors $u_1, \dots, u_K$ and the coefficients $w_{nk}$ for all $n=1\dots N, k=1\dots K$ in the database. The original vectors $x_n$ are discarded [S1].

**Formulas:**
*   Density vector definition: $x_n = [\rho_n(p_1), \rho_n(p_2), \dots, \rho_n(p_M)]^T$, where $M = m^3$ [S1].
*   Data matrix construction: $A = [x_1, x_2, \dots, x_N]$ [S1].
*   SVD decomposition: $A = U \Sigma V^*$, where $\Sigma$ is an $M \times N$ diagonal matrix with singular values $\sigma_k$ ($k=1,\dots,N$) in decreasing order, $U$ is $M \times M$, and $V$ is $N \times N$ [S1].
*   Basis selection threshold: $\sum_{k=1}^{K} \sigma_k \ge \alpha \sum_{k=1}^{N} \sigma_k$, where $\alpha$ is a user-specified constant [S1].
*   Coefficient calculation: $w_{nk} = x_n \cdot u_k$ [S1].

**Parameters:**
*   $m$: The side length of the normalized cube. Chosen experimentally to capture small-scale details of the original cloud volumes [S1].
*   $\alpha$: Threshold constant controlling approximation accuracy. Typical value: $0.9$ [S1].

**Edge Cases and Pitfalls:**
*   **Matrix Dimensions:** The method assumes $M \ge N$ [S1].
*   **Normalization Consistency:** The resizing to $m \times m \times m$ must use a consistent interpolation method to ensure that the basis vectors $u_k$ represent meaningful spatial density patterns [S1].
*   **Precision:** Floating-point precision in the SVD computation can affect the exact value of $K$ when the threshold is near a singular value boundary.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.ndimage import zoom

def svd_cloud_compression(cloud_volumes, m, alpha):
    """
    Compress a set of cloud volumes using SVD.
    
    Args:
        cloud_volumes: List of N 3D arrays representing raw cloud volumes.
        m: Integer, side length of the normalized cube.
        alpha: Float, threshold constant (e.g., 0.9).
        
    Returns:
        basis_vectors: Array of shape (K, M) containing the first K left singular vectors.
        coefficients: Array of shape (N, K) containing the projection coefficients.
        K: Integer, number of basis vectors used.
    """
    N = len(cloud_volumes)
    M = m * m * m
    
    vectors = []
    for vol in cloud_volumes:
        # [G] Extract bounding box of connected cloud region and resize to m x m x m
        normalized = normalize_volume_to_cube(vol, m)
        vectors.append(normalized.flatten())
        
    # Step 3: Construct Matrix A (M x N)
    A = np.column_stack(vectors) 
    
    # Step 4: SVD Decomposition
    U, S, Vt = np.linalg.svd(A, full_matrices=True)
    
    # Step 5: Determine K based on threshold
    total_sum = np.sum(S)
    cumulative_sum = 0.0
    K = 0
    for k in range(len(S)):
        cumulative_sum += S[k]
        if cumulative_sum >= alpha * total_sum:
            K = k + 1
            break
            
    # Step 6: Calculate Coefficients
    U_k = U[:, :K]
    coefficients = A.T @ U_k
    
    # Step 7: Return compressed data
    basis_vectors = U_k.T
    
    return basis_vectors, coefficients, K

def normalize_volume_to_cube(volume, m):
    # [G] Standard step: crop to bounding box of non-zero voxels, then resize to m x m x m
    mask = volume > 0
    if not np.any(mask):
        return np.zeros((m, m, m))
    bounds = np.where(mask)
    min_coords = [np.min(b) for b in bounds]
    max_coords = [np.max(b) for b in bounds]
    cropped = volume[min_coords[0]:max_coords[0]+1, min_coords[1]:max_coords[1]+1, min_coords[2]:max_coords[2]+1]
    scales = [m / s for s in cropped.shape]
    return zoom(cropped, scales)
```

## Operating characteristics
*   **Compression Ratio:** With $\alpha = 0.9$, the database size was reduced to 84% of the original size [S1].
*   **Approximation Error:** Under the same conditions ($\alpha = 0.9$), the approximation error was measured at 4.5% [S1].
*   **Complexity:**
    *   **Preprocessing (Training):** Dominated by the SVD of an $M \times N$ matrix. Complexity is typically $O(M N^2)$ or $O(M^2 N)$ depending on implementation, assuming $M \ge N$.
    *   **Storage:** Stores $K \times M$ basis values and $N \times K$ coefficients. Total storage $\approx K(M + N)$. Original storage is $N \times M$. Reduction factor is roughly $K/M$ if $N$ is small relative to $M$, but the source notes an 84% reduction (16% size) for $\alpha=0.9$, implying $K$ is small relative to $N$ or the specific data structure overhead is considered.
    *   **Retrieval:** Retrieval involves projecting a query vector onto the basis and comparing coefficients, which accelerates the retrieval process compared to storing and comparing full volumes [S1].
*   **Strengths:** Simple implementation; leverages the high correlation in cloud shapes; reduces storage and retrieval computational cost [S1].
*   **Limitations:** Requires a fixed resolution $m$ for all clouds, potentially losing large-scale context or wasting resolution on small clouds; sensitive to the choice of $m$ [S1].

## Background
This method was proposed in the context of a sketch-based system for realistic cloud image synthesis [S1]. It builds upon the need to store large datasets of simulated cloud volumes (tens of gigabytes) generated from atmospheric fluid dynamics simulations [Miyazaki et al. 2002]. The approach adapts standard SVD compression techniques, commonly used in image processing (e.g., JPEG), to 3D volumetric data by treating each cloud volume as a high-dimensional vector. It is specifically tailored for clouds, which have roughly similar shapes, allowing for effective low-rank approximation [S1].

## Sources
[S1] A sketch-based system for cloud volume retrieval from simulated dataset for realistic image synthesis, https://doi.org/10.1145/2817675.2817690, fetched 2026-11-05
