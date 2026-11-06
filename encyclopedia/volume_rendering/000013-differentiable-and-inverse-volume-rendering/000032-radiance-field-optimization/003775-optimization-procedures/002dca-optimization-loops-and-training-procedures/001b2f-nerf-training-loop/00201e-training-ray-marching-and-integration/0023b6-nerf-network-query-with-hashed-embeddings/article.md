# NeRF Network Query with Hashed Embeddings

Also called: HashNeRF query

## Overview
This method performs the neural network evaluation step within the NeRF training loop, mapping 3D spatial coordinates and viewing directions to density and color features using embedding functions and a network function [S1, unofficial]. It replaces expensive high-dimensional coordinate embeddings with a structured lookup process that applies position and direction embeddings before batching the concatenated features through a network function [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A batch of 3D sample points $\mathbf{x}$ and, optionally, viewing directions $\mathbf{d}$ [S1, unofficial].
*   **Outputs:** Density values $\sigma$ and color features $\mathbf{c}$ [S1, unofficial].

**Data Structures:**
*   **Embedding Functions:** `embed_fn` for position and `embeddirs_fn` for direction [S1, unofficial].

**Parameters:**
*   `netchunk`: Batch size for network evaluation to manage memory. Default: $1024 \times 64 = 65,536$ [S1, unofficial].

**Steps:**
1.  **Reshape Inputs:** Reshape input positions to a flat list with shape `[-1, inputs.shape[-1]]` [S1, unofficial].
2.  **Position Embedding:** Apply the position embedding function `embed_fn` to get embedded positions and a validity mask `keep_mask` [S1, unofficial].
3.  **Direction Embedding (Optional):** If view directions are provided, expand them to match the input shape, reshape to flat, and apply the direction embedding function `embeddirs_fn` [S1, unofficial].
4.  **Concatenation:** Concatenate position and direction embeddings along the last dimension [S1, unofficial].
5.  **Batched Network Evaluation:** Apply the network function `fn` in batches of size `netchchunk` to avoid OOM [S1, unofficial].
6.  **Masking Invalid Points:** Set sigma (density) to 0 for invalid points identified by `~keep_mask` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def query_hashnerf(positions, directions=None, embed_fn=None, embeddirs_fn=None, 
                   network_fn=None, netchunk=65536):
    """
    Query the network for density and color using hash-based embeddings.
    
    Args:
        positions: Array of shape (..., 3) containing 3D coordinates.
        directions: Array of shape (..., 3) containing viewing directions (optional).
        embed_fn: Function returning (embedded_positions, keep_mask).
        embeddirs_fn: Function returning embedded directions.
        network_fn: Function returning (sigma, features) for a batch.
        netchunk: Batch size for network evaluation.
        
    Returns:
        sigma: Density values.
        features: Color/appearance features.
    """
    # 1. Reshape input positions to flat list [-1, inputs.shape[-1]]
    original_shape = positions.shape
    positions_flat = positions.reshape(-1, positions.shape[-1])
    
    # 2. Apply position embedding function embed_fn to get embedded positions and a validity mask keep_mask
    pos_embed, keep_mask = embed_fn(positions_flat)
    
    # 3. If view directions are provided, expand them to match input shape, reshape to flat, and apply direction embedding function embeddirs_fn
    if directions is not None:
        directions_expanded = np.broadcast_to(directions, positions.shape)
        directions_flat = directions_expanded.reshape(-1, directions.shape[-1])
        dir_embed = embeddirs_fn(directions_flat)
        # 4. Concatenate position and direction embeddings along the last dimension
        input_tensor = np.concatenate([pos_embed, dir_embed], axis=-1)
    else:
        input_tensor = pos_embed
        
    # 5. Apply the network function fn in batches of size netchchunk to avoid OOM
    num_points = input_tensor.shape[0]
    sigma_list = []
    features_list = []
    for i in range(0, num_points, netchunk):
        chunk = input_tensor[i:i+netchunk]
        sigma_chunk, features_chunk = network_fn(chunk)
        sigma_list.append(sigma_chunk)
        features_list.append(features_chunk)
    sigma = np.concatenate(sigma_list, axis=0)
    features = np.concatenate(features_list, axis=0)
    
    # 6. Set sigma (density) to 0 for invalid points identified by ~keep_mask
    sigma_flat = sigma.reshape(-1)
    sigma_flat[~keep_mask] = 0.0
    sigma = sigma_flat.reshape(original_shape[:-1] + (1,))
    
    return sigma, features
```

## Subtopics

### [HybridNeRF Model Architecture](4842-hybridnerf-model-architecture.md)
HybridNeRF is a neural rendering architecture that combines multi-resolution 3D feature grids with triplane representations to predict color and signed distance fields (SDF) for efficient volumetric rendering. It distinguishes itself from its parent by replacing hash-based sparse storage with dense multi-resolution grids and triplanes, summing features across levels rather than concatenating them to reduce memory overhead. Unlike Tetrahedron Property Querying via Instant-NGP, which relies on tetrahedral meshes for property querying, HybridNeRF uses a continuous volumetric representation with a proposal network for sampling efficiency.

### [Tetrahedron Property Querying via Instant-NGP](4935-tetrahedron-property-querying-via-instant-ngp.md)
This method queries a differentiable radiance field parameterized by an Instant-NGP data structure to produce density and appearance features for individual tetrahedra in a volumetric mesh. It solves the problem of mapping point-based neural fields to volume-based primitives during optimization, where attaching features directly to vertices fails due to topological changes in the Delaunay tetrahedralization. It differs from standard NeRF point querying by using tetrahedron centers (centroid or circumcenter) and incorporating a radius-based downweighting to handle geometric degeneracy.

### Instant NGP
Represents the scene using feature vectors in a hash table and a small MLP network [S2, unofficial]. Reduces model size using two key components: Multi-resolution and Hash table [S2, unofficial].

### Stratified PDF Sampling for NeRF
A method to resample points along a ray based on the importance (weight) of previous samples, improving detail in high-density regions [S3, unofficial]. Normalize bin weights by adding epsilon (1e-5) and dividing by sum to create a PDF [S3, unofficial]. Compute the Cumulative Distribution Function (CDF) by cumsumming the PDF and padding with zeros at the start [S3, unofficial]. Generate uniform random values u in [0, 1] (or deterministic midpoints if det=True) [S3, unofficial]. Find indices in the CDF where u falls using searchsorted [S3, unofficial]. Identify lower and upper bin indices for interpolation [S3, unofficial]. Interpolate the final sample position within the bin using linear interpolation based on u's position in the CDF interval [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
The method is implemented in the HashNeRF repository [S1, unofficial] and builds on the Instant-NGP data structure [S2, unofficial].

## Sources
[S1] yashbhalgat/HashNeRF-pytorch, https://github.com/yashbhalgat/HashNeRF-pytorch, fetched 2026-10-26 (unofficial)
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S3] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
