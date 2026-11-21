# Morton Code Spatial Sorting

Also called: 3D Morton code sorting, Z-order curve sorting

## Overview
Morton Code Spatial Sorting reorders a set of 3D primitives by their spatial coordinates using a one-dimensional Morton code (Z-order curve) to maximize spatial locality in memory [S1, unofficial][S2, unofficial]. This ordering ensures that geometrically adjacent primitives are stored contiguously, which significantly improves CPU cache hit rates and reduces memory bandwidth pressure during traversal [S2, unofficial]. It sets itself apart from pure random shuffling (which destroys locality) and pure shuffled Morton orders (which balance locality with hardware load balancing) by prioritizing strict spatial coherence for compression and cache efficiency [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
- A dictionary or array of primitive parameters (e.g., `means`, `scales`, `opacities`) where each primitive $i$ has a 3D position $\mathbf{p}_i = (x_i, y_i, z_i)$ [S1, unofficial].
- The number of primitives $N$ [S1, unofficial].

**Outputs:**
- A reordered dictionary or array of primitive parameters, sorted by their Morton code [S1, unofficial].

**Data Structures:**
- `means_np`: A float32 array of shape $(N, 3)$ containing the 3D coordinates [S1, unofficial].
- `morton`: A uint32 or uint64 array of shape $(N,)$ containing the computed Morton codes [S1, unofficial].
- `order`: A uint32 array of shape $(N,)$ containing the indices that would sort `morton` in ascending order [S1, unofficial].

**Steps:**

1.  **Normalization and Quantization:**
    Calculate the bounding box of the primitives to normalize coordinates to a fixed grid [S1, unofficial].
    - Compute minimums: $\mathbf{mins} = \min(\text{means\_np}, \text{axis}=0)$ [S1, unofficial].
    - Compute maximums: $\mathbf{maxs} = \max(\text{means\_np}, \text{axis}=0)$ [S1, unofficial].
    - Compute ranges: $\mathbf{ranges} = \mathbf{maxs} - \mathbf{mins}$ [S1, unofficial].
    - **Edge Case:** If any component of $\mathbf{ranges}$ is less than $10^{-8}$, set that component to $1.0$ to avoid division by zero [S1, unofficial].
    - Normalize coordinates to $[0, 1]$: $\mathbf{normalized} = (\text{means\_np} - \mathbf{mins}) / \mathbf{ranges}$ [S1, unofficial].
    - Quantize to a 10-bit integer grid (values $0$ to $1023$):
      $$ \mathbf{quantised} = \text{clip}\left( \lfloor \mathbf{normalized} \times 1023 \rfloor, 0, 1023 \right) $$
      where $\mathbf{quantised}$ is an integer array of shape $(N, 3)$ with dtype `uint32` [S1, unofficial].

2.  **Morton Code Computation:**
    Interleave the bits of the quantized X, Y, and Z coordinates for each primitive to form a 32-bit Morton code [S1, unofficial].
    - For each primitive $i$, let $x = \mathbf{quantised}[i, 0]$, $y = \mathbf{quantised}[i, 1]$, $z = \mathbf{quantised}[i, 2]$ [S1, unofficial].
    - Compute $M_i = \text{interleave\_bits\_32}(x, y, z)$ [S1, unofficial].

3.  **Sorting:**
    - Compute the sort order indices: $\text{order} = \text{argsort}(\text{morton})$ [S1, unofficial].
    - The resulting `order` array contains the permutation indices that sort the primitives by their Morton code [S1, unofficial].

4.  **Reordering:**
    - Apply the `order` permutation to all primitive attributes in the input dictionary or array [S1, unofficial].
    - For each attribute $k$ with values $v$, the sorted values are $v[\text{order}]$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def spread(v: np.ndarray) -> np.ndarray:
    """Spread 10 bits of v into every-third-bit positions (0, 3, 6, ...)."""
    v = v.astype(np.uint64)
    v = (v | (v << 16)) & np.uint64(0x030000FF)
    v = (v | (v << 8)) & np.uint64(0x0300F00F)
    v = (v | (v << 4)) & np.uint64(0x030C30C3)
    v = (v | (v << 2)) & np.uint64(0x09249249)
    return v

def interleave_bits_32(x: np.ndarray, y: np.ndarray, z: np.ndarray) -> np.ndarray:
    """Compute Morton codes for 3D points."""
    sx = spread(x)
    sy = spread(y)
    sz = spread(z)
    return sx | (sy << 1) | (sz << 2)

def morton_sort(primitives: dict, means_np: np.ndarray) -> dict:
    """
    Sort primitives by Morton code of their means.
    
    Args:
        primitives: Dict of primitive attributes (e.g., {'means': ..., 'scales': ...}).
        means_np: Float32 array of shape (N, 3) with primitive centers.
        
    Returns:
        Sorted primitives dict.
    """
    N = means_np.shape[0]
    
    # 1. Normalize and Quantize
    mins = np.min(means_np, axis=0)
    maxs = np.max(means_np, axis=0)
    ranges = maxs - mins
    
    # Guard against zero range
    ranges = np.where(ranges < 1e-8, 1.0, ranges)
    
    normalized = (means_np - mins) / ranges
    quantised = np.clip((normalized * 1023).astype(np.uint32), 0, 1023)
    
    # 2. Compute Morton Codes
    morton = interleave_bits_32(quantised[:, 0], quantised[:, 1], quantised[:, 2])
    
    # 3. Calculate Sort Order
    order = np.argsort(morton).astype(np.uint32)
    
    # 4. Reorder Primitives
    sorted_primitives = {}
    for k, v in primitives.items():
        sorted_primitives[k] = v[order]
        
    return sorted_primitives
```

**Parameters:**
- **Grid Resolution:** 10-bit quantization is used in the reference implementation, providing $1024^3$ discrete cells [S1, unofficial].
- **Bit Width:** 32-bit integers are sufficient for 10-bit coordinates ($10 \times 3 = 30$ bits) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Range:** If all primitives share the same coordinate in one dimension, the range is zero. The method explicitly clamps ranges $< 10^{-8}$ to $1.0$ to prevent NaN/Inf during normalization [S1, unofficial].
- **Precision:** Quantization to 10 bits maps continuous coordinates to a discrete grid, which may cause distinct primitives to map to the same Morton code if they fall within the same cell [G].

## Subtopics

### [Shuffled Morton Order](10211-shuffled-morton-order.md)
Shuffled Morton Order is a vertex reordering policy for point clouds that combines Morton code sorting with batch shuffling to improve GPU cache locality and load balancing across Graphics Processing Clusters (GPCs). It solves the problem of performance instability and contention inherent in pure spatial sorting by ensuring uniform distribution of localized groups, providing high rendering performance with low sensitivity to viewpoint changes. Unlike pure Morton order which may cause excessive locality, or pure shuffled order which causes cache thrashing, this hybrid approach balances spatial coherence with hardware utilization.

### Morton-ordered Block Layout
Morton-ordered Block Layout is a spatial organization scheme that orders Gaussian blocks using Morton (Z-order) curves to preserve geometric coherence and improve CPU cache hit rates [S2, unofficial]. It maps 3D block coordinates to 1D Morton order indices to store blocks contiguously in memory [S2, unofficial]. Using Morton ordering yields a 95.2% CPU cache hit rate and 0.10 GB/iter PCIe traffic, whereas random ordering drops cache hits to 42.1% and increases traffic to 0.45 GB/iter [S2, unofficial].

### Morton Z-Order Gaussian Storage
Morton Z-Order Gaussian Storage stores Gaussians on CPU in Morton Z-order to exploit spatial coherency and improve training performance [S3, unofficial]. It stores all Gaussian properties in a single PyTorch tensor in RAM as an array of structures (AoS) [S3, unofficial]. The Gaussians are sorted in Morton Z-order on the CPU [S3, unofficial]. At every densification iteration, the Gaussians are resort to maintain the Morton Z-order [S3, unofficial].

### Entropy Encoding with Morton Order
Entropy Encoding with Morton Order is a compression step where 3D Gaussians are linearized along a space-filling curve to exploit spatial coherence of scene parameters for entropy and run-length encoding [S4]. It sorts the Gaussians in Morton order [S4]. It then applies entropy encoding and run-length encoding to the sorted data [S4].

## Operating characteristics
- **Time Complexity:** $O(N \log N)$ due to the sorting step, where $N$ is the number of primitives [G]. Bit interleaving is $O(1)$ per primitive [G].
- **Memory Complexity:** $O(N)$ to store the Morton codes and the sort order indices [G].
- **Performance:**
    - **CPU Cache Efficiency:** In out-of-core rendering scenarios, Morton-ordered block layouts achieve a 95.2% CPU cache hit rate compared to 42.1% for random ordering [S2, unofficial].
    - **Memory Bandwidth:** Morton ordering reduces PCIe traffic to 0.10 GB/iter, compared to 0.45 GB/iter for random ordering [S2, unofficial].
    - **Compression:** Sorting by Morton code improves compression ratios when quantizing and saving data (e.g., as PNG) due to increased spatial coherence [S1, unofficial].
- **Strengths:** Maximizes spatial locality, improves cache performance, and aids compression [S1, unofficial][S2, unofficial].
- **Limitations:** Sorting is a global operation, which may be costly for very large $N$ if done frequently [G].

## Background
Morton Code Spatial Sorting is derived from the general concept of Morton codes (Z-order curves) for spatial indexing [G]. In the context of 3D Gaussian Splatting, it was adopted to optimize the storage and traversal of Gaussian primitives [S3, unofficial]. The specific implementation using 10-bit quantization and bit-interleaving for sorting splats is documented in the `gsplat-mlx` repository [S1, unofficial]. The application of Morton ordering to block layouts for out-of-core optimization of billion-scale Gaussian splats is detailed in the TideGS paper [S2, unofficial]. Entropy and run-length encoding further compress the sorted data for accelerated novel view synthesis [S4].

## Sources
[S1] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
[S2] TideGS: Scalable Training of Over One Billion 3D Gaussian Splatting Primitives via Out-of-Core Optimization, https://doi.org/10.48550/arxiv.2605.20150, fetched 2026-10-26 (unofficial)
[S3] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28 (unofficial)
[S4] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
