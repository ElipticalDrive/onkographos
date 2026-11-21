# K-means Parameter Decompression
Also called: K-means Decompression
## Overview
K-means Parameter Decompression reconstructs a high-dimensional parameter tensor from compressed K-means centroids and cluster labels, effectively reversing the quantization and indexing steps of the compression phase [S1, unofficial]. It is specifically designed for Gaussian Splatting parameters, where it dequantizes integer-valued centroids back to floating-point precision and remaps them via label indices to restore the original data structure [S1, unofficial]. Unlike general vector quantization decompression, this method explicitly handles the per-dimension min/max scaling required for Gaussian Splatting attributes (such as position, opacity, and spherical harmonics coefficients) to maintain visual fidelity after clustering [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A compressed `.npz` file containing `centroids_quant` (quantized integer centroids) and `labels` (integer indices), along with metadata (`meta`) including `quantization` bit depth, `mins`/`maxs` (per-dimension bounds), `shape`, and `dtype`.
- Output: A reconstructed parameter tensor of shape `meta["shape"]` and type `meta["dtype"]`.

**Data Structures:**
- `centroids_quant`: 1D or 2D array of integers representing quantized centroid values.
- `labels`: 1D array of integers mapping each data point to a centroid index.
- `meta`: Dictionary containing:
  - `quantization`: Integer, number of bits used for quantization.
  - `mins`: Tensor/array of minimum values per dimension.
  - `maxs`: Tensor/array of maximum values per dimension.
  - `shape`: Tuple defining the target tensor shape.
  - `dtype`: Data type (e.g., `float32`).

**Steps:**
1. Load the `.npz` file to extract `centroids_quant` and `labels`.
2. Normalize the quantized centroids to the range $[0, 1]$ using the quantization bit depth:
   $$ \text{centroids\_norm} = \frac{\text{centroids\_quant}}{2^{\text{quantization}} - 1} $$
3. Scale the normalized centroids back to the original parameter range using the stored min/max bounds:
   $$ \text{centroids} = \text{centroids\_norm} \times (\text{maxs} - \text{mins}) + \text{mins} $$
4. Index the centroids array with the labels array to reconstruct the full parameter tensor:
   $$ \text{params} = \text{centroids}[\text{labels}] $$
5. Reshape the reconstructed tensor to `meta["shape"]` and cast to `meta["dtype"]`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def decompress_kmeans_params(npz_filepath, meta):
    """
    Reconstructs parameter tensor from K-means compressed data.
    
    Args:
        npz_filepath (str): Path to the .npz file containing centroids_quant and labels.
        meta (dict): Metadata dictionary with keys:
            - quantization (int): Bit depth for quantization.
            - mins (np.ndarray): Per-dimension minimum values.
            - maxs (np.ndarray): Per-dimension maximum values.
            - shape (tuple): Target tensor shape.
            - dtype (np.dtype): Target data type.
            
    Returns:
        np.ndarray: Reconstructed parameter tensor.
    """
    # Step 1: Load compressed data
    data = np.load(npz_filepath)
    centroids_quant = data['centroids_quant']
    labels = data['labels']
    
    # Step 2: Normalize quantized centroids to [0, 1]
    quantization_bits = meta['quantization']
    denominator = 2 ** quantization_bits - 1
    centroids_norm = centroids_quant / denominator
    
    # Step 3: Scale back to original range
    mins = meta['mins']
    maxs = meta['maxs']
    centroids = centroids_norm * (maxs - mins) + mins
    
    # Step 4: Index centroids with labels
    params = centroids[labels]
    
    # Step 5: Reshape and cast
    params = params.reshape(meta['shape']).astype(meta['dtype'])
    
    return params
```

**Parameters:**
- `quantization` (int): Number of bits used for centroid quantization. Determines the denominator $2^{\text{quantization}} - 1$ in the normalization step [S1, unofficial].
- `mins`, `maxs` (Tensor): Per-dimension minimum and maximum values used for linear dequantization scaling [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision Loss:** The dequantization process is lossy; the reconstructed values will differ from the original due to quantization error [S1, unofficial].
- **Boundary Conditions:** If `mins` equals `maxs` for any dimension, the denominator in scaling becomes zero, leading to division by zero or NaN values. This should be handled by checking for zero-range dimensions [G].
- **Shape Mismatch:** The product of `labels` length and the centroid dimension must match the total number of elements in `meta["shape"]` [G].

## Subtopics

### [K-means Parameter Compression](16511-k-means-parameter-compression.md)
Compresses Gaussian Splatting parameters by clustering them into centroids using K-means, then storing only cluster labels and quantized centroids to reduce memory footprint. Choose this when memory reduction is the primary goal and slight accuracy loss is acceptable.

## Operating characteristics
- **Time Complexity:** $O(N \cdot D)$, where $N$ is the number of data points (labels) and $D$ is the dimensionality of the centroids. The dominant operation is the indexing step `centroids[labels]` [S1, unofficial].
- **Memory Complexity:** $O(N + K \cdot D)$, where $K$ is the number of centroids. The output tensor requires $O(N \cdot D)$ memory [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Strengths:** Simple implementation, fast reconstruction due to direct indexing, effective for high-dimensional data with clustered structure [S1, unofficial].
- **Limitations:** Lossy compression, sensitive to quantization bit depth, requires storing min/max bounds [S1, unofficial].
- **When to Use:** When decompressing Gaussian Splatting parameters stored in K-means compressed format [S1, unofficial].
- **When Not to Use:** When lossless reconstruction is required or when data does not exhibit strong clustering properties [G].

## Background
K-means Parameter Decompression originates from the CLOD-3DGS project by Facebook Research, which focuses on compressing 3D Gaussian Splatting parameters for efficient storage and transmission [S1, unofficial]. The method leverages the fact that Gaussian Splatting parameters often exhibit clusterable structures, allowing significant compression with minimal visual impact [S1, unofficial]. It is used in conjunction with K-means Parameter Compression to enable efficient rendering of large-scale Gaussian Splatting scenes [S1, unofficial].

## Sources
[S1] facebookresearch/CLOD-3DGS, https://github.com/facebookresearch/CLOD-3DGS, fetched 2026-10-27 (unofficial)
