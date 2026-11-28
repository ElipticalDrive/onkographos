# Sparse High Resolution Volume
Also called: Multi-stage Sparse Encoding, Sparse Volume Optimization

## Overview
Sparse High Resolution Volume is a memory-efficient encoding method that replaces dense high-resolution feature volumes with a sparse set of valid voxels located near the surface, determined by a coarse surface reconstruction [S1, unofficial]. It sets itself apart from its sibling [Hierarchical Volume Encoding](20354-hierarchical-volume-encoding.md) by introducing a two-phase optimization strategy: first optimizing a coarse hierarchical structure, then sparsifying the highest-resolution layers based on that coarse geometry to drastically reduce memory consumption while maintaining high-frequency detail [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A coarse surface reconstruction $S_c$ obtained from the previous stage of Hierarchical Volume Encoding; a target high-resolution grid size $N$ (e.g., $512^3$ or $1024^3$); and a feature dimension $d$ (typically 4).
*   **Output:** A sparse embedding table $T_e$ containing features for valid voxels, and an index table $T_i$ mapping grid coordinates to $T_e$ indices.

**Data Structures**
1.  **Index Table ($T_i$):**
    *   **Layout:** A 1D array of integers.
    *   **Size:** $N^3$, where $N$ is the resolution of the high-resolution volume (e.g., $256^3$ for the lookup table size in stages 2 and 3 [S1, unofficial]).
    *   **Content:** Stores indices into $T_e$. Initialized to $-1$.
2.  **Embedding Table ($T_e$):**
    *   **Layout:** A 1D array of vectors (embeddings).
    *   **Size:** $n + 1$, where $n$ is the number of valid voxels ($n \ll N^3$).
    *   **Content:** Stores the actual feature vectors. The last element (index $n$) serves as a default/fallback value for invalid voxels [S1, unofficial].

**Parameters**
*   **Stage 2 Sparse Resolution:** $512 \times 512 \times 512$ with feature dimension 4 [S1, unofficial].
*   **Stage 3 Sparse Resolution:** $1024 \times 1024 \times 1024$ with feature dimension 4 [S1, unofficial].
*   **Index Table Length:** $256^3$ for both Stage 2 and Stage 3 sparse volumes [S1, unofficial].

**Algorithm Steps**

1.  **Coarse Reconstruction (Stage 1):** Optimize the basic hierarchical structure (max resolution 256) to obtain a coarse surface reconstruction $S_c$ [S1, unofficial].
2.  **Surface Dilation:** Dilate the coarse surface $S_c$ to include surrounding voxels, ensuring all valid voxels near the surface are captured [S1, unofficial].
3.  **Valid Voxel Identification:** Identify the set of valid voxels $V_{h}^{valid}$ in the high-resolution volume based on the dilated surface. Voxels far from the surface are culled [S1, unofficial].
4.  **Index Table Initialization:** Initialize $T_i$ of size $N^3$ with all values set to $-1$ [S1, unofficial].
5.  **Mapping Valid Voxels:** For each valid voxel $v_j \in V_{h}^{valid}$ with integer coordinates $(x, y, z)$:
    *   Compute the linear grid index: $idx = x + y \cdot N + z \cdot N^2$.
    *   Assign a new unique index $j$ (from $0$ to $n-1$) to this voxel.
    *   Set $T_i[idx] = j$.
6.  **Embedding Table Construction:** Create $T_e$ of size $n+1$. Populate the first $n$ entries with the embeddings for the valid voxels. Set the last entry ($T_e[n]$) to a default/fallback embedding value [S1, unofficial].

**Lookup Procedure**
Given a floating-point 3D position $\mathbf{x} = (x, y, z)$:
1.  **Round Coordinates:** Compute the integer coordinates of the 8 surrounding voxels by rounding $x, y, z$ to the nearest integers. Let these be $(x_i, y_i, z_i)$ for $i \in \{0, \dots, 7\}$.
2.  **Compute Grid Indices:** For each surrounding voxel, compute its linear index in the $N^3$ grid:
    $$ f(x_i, y_i, z_i) = x_i + y_i \cdot N + z_i \cdot N^2 $$
    where $N$ is the volume resolution [S1, unofficial].
3.  **Retrieve Embedding Indices:** Look up the index in $T_i$:
    $$ j_i = T_i[f(x_i, y_i, z_i)] $$
4.  **Handle Invalid Voxels:** If $j_i = -1$, set $j_i = n$ (the fallback index) [S1, unofficial].
5.  **Fetch Embeddings:** Retrieve the embeddings $e_i = T_e[j_i]$ for all 8 surrounding voxels.
6.  **Trilinear Interpolation:** Fuse the 8 embeddings $e_i$ using trilinear interpolation based on the fractional parts of $\mathbf{x}$ to produce the final encoding for $\mathbf{x}$ [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class SparseHighResVolume:
    def __init__(self, resolution: int, feature_dim: int, num_valid_voxels: int):
        """
        Initialize the sparse high-resolution volume.
        
        Args:
            resolution: N, the side length of the cubic volume (e.g., 512).
            feature_dim: d, the dimension of each embedding vector.
            num_valid_voxels: n, the number of valid voxels near the surface.
        """
        self.N = resolution
        self.d = feature_dim
        self.n = num_valid_voxels
        
        # T_i: Index table, size N^3, initialized to -1
        self.T_i = np.full((N**3,), -1, dtype=np.int32)
        
        # T_e: Embedding table, size n+1, last entry is fallback
        self.T_e = np.zeros((num_valid_voxels + 1, feature_dim), dtype=np.float32)
        # Initialize fallback embedding (e.g., zeros or learned constant)
        self.T_e[-1] = 0.0 

    def set_valid_voxels(self, valid_coords: np.ndarray, embeddings: np.ndarray):
        """
        Populate the sparse volume with valid voxels.
        
        Args:
            valid_coords: Array of shape (n, 3) containing integer coordinates (x, y, z)
                          of the valid voxels.
            embeddings: Array of shape (n, d) containing the feature vectors for
                        the valid voxels.
        """
        n = valid_coords.shape[0]
        if n != self.n:
            raise ValueError("Number of valid voxels must match initialization.")
        
        for i in range(n):
            x, y, z = valid_coords[i]
            # Compute linear index in the N^3 grid
            idx = x + y * self.N + z * (self.N ** 2)
            
            # Map to index in T_e
            self.T_i[idx] = i
            
            # Store embedding in T_e
            self.T_e[i] = embeddings[i]

    def lookup(self, x: float, y: float, z: float) -> np.ndarray:
        """
        Retrieve the encoding for a floating-point position using trilinear interpolation.
        
        Args:
            x, y, z: Floating-point coordinates.
            
        Returns:
            The interpolated embedding vector of shape (d,).
        """
        # 1. Round to get 8 surrounding integer coordinates
        # We need the floor and ceil for each dimension to define the 8 corners
        x0, x1 = int(np.floor(x)), int(np.ceil(x))
        y0, y1 = int(np.floor(y)), int(np.ceil(y))
        z0, z1 = int(np.floor(z)), int(np.ceil(z))
        
        # Fractional parts for interpolation weights
        fx = x - x0
        fy = y - y0
        fz = z - z0
        
        # 2. Compute linear indices for all 8 corners
        # Note: Coordinates might be out of bounds if x,y,z are outside [0, N-1]
        # In practice, clamping or boundary handling is required. 
        # Assuming valid input range for this implementation.
        corners = [
            (x0, y0, z0), (x1, y0, z0),
            (x0, y1, z0), (x1, y1, z0),
            (x0, y0, z1), (x1, y0, z1),
            (x0, y1, z1), (x1, y1, z1)
        ]
        
        embeddings = []
        for cx, cy, cz in corners:
            # Compute grid index
            idx = cx + cy * self.N + cz * (self.N ** 2)
            
            # Retrieve index from T_i
            j = self.T_i[idx]
            
            # If -1, use fallback (last element)
            if j == -1:
                j = self.n
            
            embeddings.append(self.T_e[j])
            
        embeddings = np.array(embeddings)
        
        # 3. Trilinear Interpolation
        # Weights for the 8 corners
        # w000 = (1-fx)(1-fy)(1-fz)
        # w100 = fx(1-fy)(1-fz)
        # ... etc.
        
        w000 = (1 - fx) * (1 - fy) * (1 - fz)
        w100 = fx * (1 - fy) * (1 - fz)
        w010 = (1 - fx) * fy * (1 - fz)
        w110 = fx * fy * (1 - fz)
        w001 = (1 - fx) * (1 - fy) * fz
        w101 = fx * (1 - fy) * fz
        w011 = (1 - fx) * fy * fz
        w111 = fx * fy * fz
        
        weights = np.array([w000, w100, w010, w110, w001, w101, w011, w111])
        
        # Dot product of weights and embeddings
        result = np.dot(weights, embeddings)
        
        return result
```

**Edge Cases and Pitfalls**
*   **Boundary Conditions:** If the query position $\mathbf{x}$ lies outside the $[0, N-1]$ range, the rounded coordinates may produce indices outside $T_i$. The source does not specify boundary handling (e.g., clamping, wrapping, or returning fallback). Implementers should clamp coordinates to $[0, N-1]$ or handle out-of-bounds indices by returning the fallback embedding [S1, unofficial].
*   **Precision:** The mapping function $f(x)$ uses integer arithmetic. Ensure that $N^3$ fits within the integer type used for $T_i$ (e.g., $1024^3 \approx 10^9$, which fits in a standard 32-bit signed integer, but close to the limit).
*   **Degenerate Input:** If no valid voxels are found ($n=0$), $T_e$ will only contain the fallback value. All lookups will return the fallback embedding.

## Subtopics

### Hierarchical Volume Encoding
The parent method that provides the coarse surface reconstruction $S_c$ used to cull voxels. It uses multiple 3D feature volumes at different resolutions optimized jointly with MLPs. [Link](20354-hierarchical-volume-encoding.md)

## Operating characteristics
*   **Memory Complexity:** The memory usage is dominated by $T_e$ (size $n+1$) and $T_i$ (size $N^3$). Since $n \ll N^3$, this is significantly more memory-efficient than a dense volume of size $N^3 \times d$. For example, a $512^3$ dense volume with dimension 4 would require $512^3 \times 4 \times 4$ bytes $\approx 2$ GB, whereas the sparse version only stores $n$ embeddings plus the index table [S1, unofficial].
*   **Time Complexity:** Lookup is $O(1)$ per voxel, involving a constant number of arithmetic operations and table lookups. The overhead of computing the 8 surrounding indices and interpolating is small.
*   **Performance:** The method allows for high-resolution details (e.g., $1024^3$) to be encoded without prohibitive memory costs, enabling sharper surface reconstruction compared to lower-resolution dense volumes. It is used in conjunction with rendering losses ($L_{color}$) and Eikonal losses ($L_{eik}$) [S1, unofficial].
*   **Strengths:** Significant memory savings for high-resolution volumes; enables fine detail capture.
*   **Limitations:** Requires a prior coarse reconstruction to determine valid voxels; cannot represent features in regions far from the surface (which are culled); lookup performance depends on cache efficiency of $T_i$ and $T_e$.

## Background
Sparse High Resolution Volume was introduced in the paper "HIVE: HIerarchical Volume Encoding for Neural Implicit Surface Reconstruction" [S1, unofficial]. It addresses the problem that dense high-resolution volumes (e.g., $256^3$ or larger) consume excessive memory, much of which is wasted on unoccupied space. By adopting a multi-stage optimization scheme, the method first obtains a coarse surface and then sparsifies the high-resolution layers, retaining only voxels near the surface. This approach is part of the broader family of neural implicit surface reconstruction methods, building on techniques like NeuS and NeuralWarp [S1, unofficial].

## Sources
[S1] HIVE: HIerarchical Volume Encoding for Neural Implicit Surface Reconstruction, https://doi.org/10.48550/arxiv.2408.01677, fetched 2026-10-26 (unofficial)
