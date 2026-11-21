# K-Nearest Neighbors Query

Also called: KNNQuery

## Overview
A method to find the $k$ nearest neighbors for each point in a target set within a source set, returning indices and squared distances. It performs K-Nearest Neighbors search on 3D points using a CUDA kernel, handling batched inputs with offsets. It differs from [Efficient Local kNN](10975-efficient-local-knn.md) by focusing on global batched queries with explicit offset management, and from [PointOps2 CUDA Kernels](13912-pointops2-cuda-kernels.md) by being a specific high-level API wrapper rather than a low-level kernel library.

## How it works
**Inputs:**
- `src`: Source point cloud coordinates, shape $(B, M, 3)$ or $(B, 3, M)$ depending on `transpose`.
- `x`: Query point cloud coordinates, shape $(B, N, 3)$ or $(B, 3, N)$ depending on `transpose`.
- `src_offset`: Cumulative offsets for the source set, shape $(B+1)$.
- `x_offset`: Cumulative offsets for the query set, shape $(B+1)$.
- `k`: Number of neighbors to find.
- `transpose`: Boolean flag indicating if input tensors are in $(B, 3, N)$ format.

**Outputs:**
- `idx`: Indices of the $k$ nearest neighbors in the source set, shape $(B, N, k)$, type `long`.
- `dist`: Euclidean distances to the neighbors, shape $(B, N, k)$.

**Data Structures:**
- The offsets arrays `src_offset` and `x_offset` define the start index of each batch element in the flattened 2D representation. For batch $b$, the points are located in the range `[offset[b], offset[b+1])`.

**Steps:**
1. If the query set `x` is `None`, default to using `src` as the query set.
2. If `transpose` is true, transpose `x` and `src` from $(B, N, 3)$ to $(B, 3, N)$ and make them contiguous.
3. Extract batch size $B$, query count $N$, and source count $M$.
4. Reshape `x` to $(B \times N, 3)$ and `src` to $(B \times M, 3)$.
5. Compute cumulative offsets `x_offset` and `src_offset` for the batched CUDA kernel call.
6. Call the CUDA kernel `knnquery(k, src, x, src_offset, x_offset)` to get indices `idx_raw` and squared distances `dist2`.
7. Adjust indices by subtracting `(src_offset - M)` to correct for batch offsets.
8. Reshape `idx` to $(B, N, k)$ and return as long integers.
9. Return `idx` and the square root of `dist2`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import pointops_cuda # Assuming this module contains knnquery_cuda

def knnquery(k, src, x, src_offset, x_offset, transpose=False):
    """
    k: int, number of neighbors
    src: Tensor (B, M, 3) or (B, 3, M)
    x: Tensor (B, N, 3) or (B, 3, N)
    src_offset: Tensor (B+1), cumulative offsets for src
    x_offset: Tensor (B+1), cumulative offsets for x
    transpose: bool, if True inputs are (B, 3, N)
    """
    if x is None:
        x = src.clone()
        
    if transpose:
        src = src.transpose(1, 2).contiguous()
        x = x.transpose(1, 2).contiguous()
        
    B, N, _ = x.shape
    _, M, _ = src.shape
    
    # Reshape to 2D for CUDA kernel
    src_flat = src.view(-1, 3) # (B*M, 3)
    x_flat = x.view(-1, 3)     # (B*N, 3)
    
    # Compute offsets if not provided or verify them
    # Note: In practice, offsets are usually pre-computed. 
    # Here we assume they are passed as arguments.
    
    # Call CUDA kernel
    # Returns idx_raw (B*N, k) and dist2 (B*N, k)
    idx_raw, dist2 = pointops_cuda.knnquery_cuda(k, src_flat, x_flat, src_offset, x_offset)
    
    # Adjust indices: subtract the base offset of the source for each batch
    # idx_raw is in the flattened space [0, B*M)
    # We need to map it back to local indices [0, M) or keep global if required.
    # The note says: "Adjust indices by subtracting (src_offset - m)"
    # This implies idx_raw includes the offset. 
    # Let's assume idx_raw is global index in src_flat.
    # To get local index: idx_local = idx_raw - src_offset[batch_id]
    
    # Reshape idx_raw to (B, N, k)
    idx_raw = idx_raw.view(B, N, k)
    
    # Create a tensor of batch indices for each query point
    # batch_idx has shape (B, N, k) where each element is the batch ID
    batch_indices = torch.arange(B).view(B, 1, 1).expand(B, N, k).to(idx_raw.device)
    
    # Get the source offset for each query point's batch
    # src_offset[batch_idx] gives the start index for that batch in src_flat
    src_offsets_for_queries = src_offset[batch_indices]
    
    # Adjust indices
    idx_local = idx_raw - src_offsets_for_queries
    
    # Return indices and distances
    dist = torch.sqrt(dist2)
    
    return idx_local.long(), dist
```

**Parameters:**
- `k`: Typical values depend on the application density, e.g., 16, 32, 64.
- `transpose`: False for standard $(B, N, 3)$ layout.

**Edge Cases:**
- If `x` is `None`, it defaults to `src`.
- Precision issues may arise in squared distance calculations for very large coordinates; use double precision if necessary [G].
- Boundary conditions: Ensure `src_offset` and `x_offset` are correctly computed to avoid out-of-bounds access in the CUDA kernel.

## Subtopics

### N-th Closest (Worley/Hart)
A basis function introduced by Worley and Hart [S2, book] that places 'feature points' at random locations in $\mathbb{R}^3$ and calculates the distance from a surface point to each of the $n$ closest feature points. It is used to index a color spline, adding a flagstone style texture to the surface of the object. It is chosen when procedural texturing with cellular noise patterns is required.

## Operating characteristics
- **Time Complexity:** $O(N \cdot M \cdot \log M)$ or $O(N \cdot M)$ depending on the underlying spatial data structure (e.g., KD-Tree vs. brute force). The CUDA implementation optimizes this for parallel execution.
- **Memory Complexity:** $O(N \cdot k)$ for storing indices and distances.
- **Measured Performance:** High performance on GPU due to parallelization. Specific metrics depend on hardware and resolution.
- **Strengths:** Efficient batched processing, GPU acceleration.
- **Limitations:** Requires CUDA-capable hardware. Memory usage scales with batch size and query count.
- **When to use:** When fast neighbor search is needed for large point clouds in a batched manner.
- **When not to use:** On CPU-only systems or when memory is extremely constrained.

## Background
Originated from the need for efficient neighbor search in point cloud processing, particularly in the context of 4D Gaussian Splatting and other neural rendering techniques. The specific implementation is found in the `4d-gaussian-splatting` repository [S1, unofficial].

## Sources
[S1] LogosIntelligenceLab/4d-gaussian-splatting, https://github.com/LogosIntelligenceLab/4d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
