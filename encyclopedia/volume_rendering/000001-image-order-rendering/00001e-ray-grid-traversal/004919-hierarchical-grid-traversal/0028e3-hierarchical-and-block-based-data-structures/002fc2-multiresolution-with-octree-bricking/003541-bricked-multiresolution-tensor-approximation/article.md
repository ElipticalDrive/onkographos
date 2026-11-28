# Bricked Multiresolution Tensor Approximation

Also called: Bricked Multiresolution TA, Multiresolution TA octree

## Overview
Bricked Multiresolution Tensor Approximation is a data management and rendering technique for interactive visualization of large volumetric datasets [S1]. It solves the problem of limited working memory by dividing the volume into small cubical bricks, approximating each brick using rank-reduced Tucker decomposition, and organizing them into an out-of-core octree structure [S1]. It sets itself apart from sibling methods by combining offline tensor-based compression with an adaptive, view-dependent loader that maintains a GPU cache of reconstructed bricks, enabling stack-less ray-casting with on-demand reconstruction only upon cache misses [S1].

## How it works
**Inputs:** Original volumetric dataset (scalar field); Brick width $B$; Rank reduction parameter $R$; Reconstruction rank $K$ (optional, for interactive control) [S1].
**Outputs:** Out-of-core octree structure containing quantized tensor decompositions; Rendered image via CUDA ray-caster [S1].

**Data Structures:**
1.  **Brick:** A cubical subvolume of size $B^3$ [S1].
    *   **Core Tensor ($\mathcal{B}$):** A third-order tensor of size $R \times R \times R$ [S1].
    *   **Factor Matrices ($U^{(1)}, U^{(2)}, U^{(3)}$):** Three matrices, each of size $B \times R$ [S1].
    *   **Binary Histogram:** A 64-bit integer used for transfer-function-based culling [S1].
    *   **Quantization:** The tensor and factor matrix coefficients are quantized to reduce memory usage [S1].
2.  **Octree Node:** Represents a brick at a specific resolution level [S1]. Contains pointers to children (if non-leaf) or the brick data (if leaf) [S1]. Includes neighbor pointers for spatial indexing [S1].
3.  **GPU Cache:** A 3D texture storing recently used, reconstructed bricks [S1].

**Parameters:**
*   **Brick Width ($B$):** Set to $32$ voxels [S1]. The source formula $B = (2^8 + 2 + 2) = 32$ contains a typographical error ($2^8=256$), but the text explicitly defines $B=32$ with a $2$-voxel overlap at each boundary for trilinear interpolation and gradient computation [S1].
*   **Rank Reduction ($R$):** Typically $R = B/2$ [S1]. For $B=32$, $R=16$, resulting in a rank-$(R, R, R)$ Tucker Approximation [S1].
*   **Reconstruction Rank ($K$):** User-selectable integer $K \in [1, R]$ [S1]. Allows interactive trade-off between detail and performance [S1].

**Preprocessing (Offline):**
The preprocessing is performed in a low-memory setting using a bottom-up process on a brick-by-brick basis [S1].
1.  **Leaf Bricks:** Constructed by sampling the original dataset [S1]. Each leaf brick $A \in \mathbb{R}^{B \times B \times B}$ is approximated using a rank-reduced Tucker decomposition [S1].
2.  **Tucker Decomposition:** For a brick $A$, the decomposition is defined as:
    $$ A \approx \mathcal{B} \times_1 U^{(1)} \times_2 U^{(2)} \times_3 U^{(3)} $$
    where $\mathcal{B}$ is the core tensor of size $R \times R \times R$, and $U^{(n)}$ are the factor matrices of size $B \times R$ [S1].
3.  **Quantization:** The coefficients of $\mathcal{B}$ and $U^{(n)}$ are quantized to reduce storage size [S1].
4.  **Non-Leaf Bricks:** Constructed from their eight previously constructed children [S1]. The children are dequantized and reconstructed into full $B^3$ volumes [S1]. These volumes are spatially averaged to form the parent brick [S1]. The parent brick is then approximated with a new rank-reduced Tucker decomposition [S1].
5.  **Histogram:** A 64-bit binary histogram is computed for each brick and stored alongside the decomposition [S1].
6.  **Octree Assembly:** Bricks are organized into an octree, where each level represents a coarser resolution [S1]. The root represents the coarsest level [S1].

**Runtime (Adaptive Loading & Rendering):**
1.  **Working Set Maintenance:** An adaptive loader updates a view- and transfer-function-dependent working set of bricks [S1]. It asynchronously fetches data from the out-of-core structure and maintains the working set on CPU and GPU memory [S1].
2.  **GPU Cache:** The loader maintains a cache of recently used volume bricks on the GPU, stored in a 3D texture [S1].
3.  **Spatial Index:** At each frame, the loader constructs a spatial index for the current working set in the form of an octree with neighbor pointers [S1].
4.  **Ray-Casting:** A CUDA stack-less octree ray-caster traverses the octree [S1]. It uses pre-integrated scalar transfer functions and supports various shading modes [S1].
5.  **Reconstruction on Miss:** The ray-caster works on reconstructed bricks [S1]. Reconstruction steps occur only upon GPU cache misses [S1]. When a cache miss occurs, the quantized tensor decomposition is dequantized and reconstructed on demand by the adaptive loader on the GPU [S1].
6.  **Interactive Rank Control:** The user can change the reconstruction rank $K$ during visualization [S1]. The reconstruction uses only the $K$ most significant ranks of the tensor decomposition [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def quantize(tensor):
    """Quantize tensor coefficients to reduce memory usage [G]."""
    # [G] Exact quantization scheme is not provided in the source.
    # Standard practice rounds to a fixed-point format or clips to a range.
    return np.round(tensor, decimals=2)

def dequantize(quantized_tensor):
    """Restore quantized tensor coefficients [G]."""
    # [G] Inverse of quantization; typically identity or scaling.
    return quantized_tensor

def compute_64bit_histogram(brick_data):
    """Compute a 64-bit binary histogram for transfer-function culling [G]."""
    # [G] Source does not specify binning strategy.
    # Standard approach: map scalar values to 64 bins, set bit if occupied.
    min_val, max_val = brick_data.min(), brick_data.max()
    bins = np.linspace(min_val, max_val, 65)
    hist, _ = np.histogram(brick_data, bins=bins)
    return np.packbits(hist > 0).astype(np.uint64)

def tucker_decomposition(brick_data, rank):
    """
    Computes rank-reduced Tucker decomposition for a brick.
    brick_data: numpy array of shape (B, B, B)
    rank: int, R
    Returns: core_tensor (R, R, R), factor_matrices [U1 (B, R), U2 (B, R), U3 (B, R)]
    """
    # [G] Standard Higher-Order SVD (HOSVD) or iterative ALS method.
    # Mode-1 unfolding
    U1, _, _ = np.linalg.svd(brick_data.reshape(B, -1), full_matrices=False)
    U1 = U1[:, :rank]
    core = np.tensordot(U1.T, brick_data, axes=1)
    
    # Mode-2 unfolding
    U2, _, _ = np.linalg.svd(core.reshape(B, -1), full_matrices=False)
    U2 = U2[:, :rank]
    core = np.tensordot(U2.T, core, axes=1)
    
    # Mode-3 unfolding
    U3, _, _ = np.linalg.svd(core.reshape(B, -1), full_matrices=False)
    U3 = U3[:, :rank]
    core = np.tensordot(U3.T, core, axes=1)
    
    return core, [U1, U2, U3]

def reconstruct_tucker(core, factors, rank_k):
    """
    Reconstructs the brick from Tucker decomposition using only the top K ranks.
    core: numpy array of shape (R, R, R)
    factors: list of 3 matrices, each (B, R)
    rank_k: int, K <= R
    Returns: numpy array of shape (B, B, B)
    """
    # Truncate factors and core to rank K
    U1 = factors[0][:, :rank_k]
    U2 = factors[1][:, :rank_k]
    U3 = factors[2][:, :rank_k]
    core_k = core[:rank_k, :rank_k, :rank_k]
    
    # Reconstruct: A = B x1 U1 x2 U2 x3 U3
    temp1 = np.tensordot(U1, core_k, axes=([1], [0])) # Shape (B, R, R)
    temp2 = np.tensordot(U2, temp1, axes=([1], [1])) # Shape (B, B, R)
    result = np.tensordot(U3, temp2, axes=([1], [2])) # Shape (B, B, B)
    
    return result

def preprocess_leaf(leaf_data, B, R):
    """Preprocesses a leaf brick."""
    core, factors = tucker_decomposition(leaf_data, R)
    quantized_core = quantize(core)
    quantized_factors = [quantize(f) for f in factors]
    histogram = compute_64bit_histogram(leaf_data)
    return {'core': quantized_core, 'factors': quantized_factors, 'histogram': histogram, 'is_quantized': True}

def preprocess_non_leaf(children_bricks, B, R):
    """Preprocesses a non-leaf brick from its 8 children."""
    child_volumes = []
    for child in children_bricks:
        core = dequantize(child['core'])
        factors = [dequantize(f) for f in child['factors']]
        vol = reconstruct_tucker(core, factors, R)
        child_volumes.append(vol)
    
    # [G] Source states "spatially averaged". 
    # Implementation assumes element-wise mean of the 8 child volumes.
    combined_volume = np.mean(child_volumes, axis=0)
    core, factors = tucker_decomposition(combined_volume, R)
    quantized_core = quantize(core)
    quantized_factors = [quantize(f) for f in factors]
    histogram = compute_64bit_histogram(combined_volume)
    return {'core': quantized_core, 'factors': quantized_factors, 'histogram': histogram, 'is_quantized': True}

def render_ray_caster(octree, ray, transfer_function, gpu_cache):
    """CUDA stack-less octree ray-caster."""
    # Traverse octree spatial index
    # Check GPU cache (3D texture) for current brick
    # If cache miss: dequantize & reconstruct on GPU
    # Integrate along ray using pre-integrated scalar transfer function
    pass
```

## Operating characteristics
*   **Time Complexity:** Preprocessing is $O(N \cdot B^3)$ for $N$ bricks, with tensor decomposition overhead [S1]. Runtime complexity depends on the number of cache hits [S1]. Reconstruction on cache miss is $O(B^3 \cdot R^3)$ or similar, depending on the reconstruction method [S1]. Ray-casting is $O(\text{ray length} \cdot \text{samples})$ [S1].
*   **Memory Complexity:** Significantly reduced compared to raw data due to Tucker decomposition and quantization [S1]. The octree structure adds overhead [S1]. The adaptive loader maintains a working set on CPU and GPU memory, asynchronously fetching from the out-of-core structure [S1].
*   **Performance:** Enables interactive visualization of large datasets [S1]. Performance scales with the number of cache hits [S1]. Higher $K$ values increase reconstruction time and detail [S1].
*   **Strengths:** Low memory footprint, adaptive loading, interactive rank control [S1].
*   **Limitations:** Reconstruction latency on cache misses, quantization artifacts [S1].

## Background
The method was introduced in the paper "Interactive Multiscale Tensor Reconstruction for Multiresolution Volume Visualization" [S1]. It builds on the MOVR approach for adaptive refinement [S1]. It is used for interactive visualization of large volumetric datasets [S1].

## Sources
[S1] Interactive Multiscale Tensor Reconstruction for Multiresolution Volume ..., https://ics.uci.edu/~gopi/SamplePubs/MultiscaleTensorVis.pdf, fetched 2026-10-26
