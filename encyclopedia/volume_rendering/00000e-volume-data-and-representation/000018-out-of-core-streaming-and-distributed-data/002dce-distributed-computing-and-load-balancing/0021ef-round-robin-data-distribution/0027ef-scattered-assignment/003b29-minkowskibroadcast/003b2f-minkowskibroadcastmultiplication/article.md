# MinkowskiBroadcastMultiplication

Also called: BroadcastMultiplication, MinkowskiBroadcastBase with BroadcastMode.ELEMENTWISE_MULTIPLICATION

## Overview
MinkowskiBroadcastMultiplication broadcasts a reduced global feature vector to all active coordinates of a sparse tensor by multiplying it element-wise with the input features [S1, S2, unofficial]. It solves the problem of applying global context (e.g., from global pooling) to local sparse features without changing the coordinate structure [S1, S2, unofficial]. It differs from [MinkowskiBroadcastAddition](8928-minkowskibroadcastaddition.md) (which adds the global vector) and [MinkowskiBroadcastConcatenation](8933-minkowskibroadcastconcatenation.md) (which concatenates the global vector to the input features) by performing element-wise multiplication [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A `SparseTensor` with features $\mathbf{x}_1$ and a coordinate map key.
- **Input Glob:** A `SparseTensor` with features $\mathbf{x}_2$ reduced to the origin (typically via global pooling).
- **Output:** A `SparseTensor` with the same coordinate map key as the input, containing the multiplied features.

**Data Structures:**
- `SparseTensor`: Contains features (tensor of shape $[N, C]$ where $N$ is number of active coordinates and $C$ is channels) and a `coordinate_map_key`.
- `CoordinateManager`: Manages coordinate maps and batch indices.
- `BroadcastMode`: Enum constant `ELEMENTWISE_MULTIPLICATION`.

**Numbered Steps:**
1. Initialize the operation with `BroadcastMode.ELEMENTWISE_MULTIPLICATION`.
2. Retrieve the batch indices and row offsets for the input coordinates from the `coordinate_manager` using the input's `coordinate_map_key`.
3. Allocate an output feature tensor of the same size as the input features.
4. For each batch index $b$ and corresponding row offsets `rows`:
   - Extract the global feature vector $\mathbf{x}_2$ for batch $b$ from `input_glob.F`.
   - Assign $\mathbf{x}_2$ to all positions in the output feature tensor corresponding to `rows`.
5. Multiply the assigned global features element-wise with the input features $\mathbf{x}_1$ at those positions.
6. Return a new `SparseTensor` with the computed features and the input's `coordinate_map_key`.

**Formula:**
$$ \mathbf{y}_\mathbf{u} = \mathbf{x}_{1, \mathbf{u}} \times \mathbf{x}_2 \quad \text{for} \quad \mathbf{u} \in \mathcal{C}^\text{in} $$
Where:
- $\mathbf{y}_\mathbf{u}$ is the output feature vector at coordinate $\mathbf{u}$.
- $\mathbf{x}_{1, \mathbf{u}}$ is the input feature vector at coordinate $\mathbf{u}$.
- $\mathbf{x}_2$ is the global feature vector (reduced to the origin).
- $\times$ denotes element-wise multiplication.
- $\mathcal{C}^\text{in}$ is the set of input coordinates.

**Parameters:**
- `operation_type`: Set to `BroadcastMode.ELEMENTWISE_MULTIPLICATION` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def minkowski_broadcast_multiplication(input_sparse_tensor, input_glob_sparse_tensor, coordinate_manager):
    """
    Broadcasts reduced global features to all input coordinates by multiplying them element-wise.
    
    Args:
        input_sparse_tensor: SparseTensor with features x1 and coordinate_map_key.
        input_glob_sparse_tensor: SparseTensor with features x2 reduced to the origin.
        coordinate_manager: CoordinateManager instance.
        
    Returns:
        SparseTensor with output features and input coordinate_map_key.
    """
    # Extract input features and coordinate map key
    x1 = input_sparse_tensor.F
    coord_map_key = input_sparse_tensor.coordinate_map_key
    device = x1.device
    
    # Get batch indices and row offsets for the input coordinates
    batch_indices, batch_rows = coordinate_manager.origin_map(coord_map_key)
    
    # Get global features
    x2_glob = input_glob_sparse_tensor.F
    
    # Initialize output features with zeros (or copy x1 to preserve shape)
    # The output features will be x1 * broadcasted x2
    n_input = x1.shape[0]
    n_channels = x2_glob.shape[1]
    y = torch.zeros(n_input, n_channels, device=device, dtype=x1.dtype)
    
    # Iterate over batches
    for b, rows in zip(batch_indices, batch_rows):
        # Extract global feature vector for this batch
        # x2_glob is assumed to be shaped [num_batches, n_channels]
        global_feat = x2_glob[b]
        
        # Broadcast global_feat to all input coordinates in this batch
        # and multiply element-wise with input features
        y[rows] = x1[rows] * global_feat
        
    # Return new SparseTensor with output features and input coordinate map key
    return SparseTensor(y, coordinate_map_key=coord_map_key, coordinate_manager=coordinate_manager)
```

**Edge Cases and Pitfalls:**
- **Dimension Mismatch:** The number of channels in `input_glob_sparse_tensor` must match the number of channels in `input_sparse_tensor` [S1, S2, unofficial].
- **Empty Input:** If the input sparse tensor has no active coordinates, the output should also be empty [G].
- **Batch Indexing:** Ensure that `batch_indices` and `batch_rows` correctly map to the global features, which are typically reduced to the origin per batch [S1, S2, unofficial].

## Subtopics

### [MinkowskiBroadcastAddition](8928-minkowskibroadcastaddition.md)
Adds the global feature vector to the input features. Choose when additive context is desired.

### [MinkowskiBroadcastConcatenation](8933-minkowskibroadcastconcatenation.md)
Concatenates the global feature vector to the input features. Choose when preserving both local and global features separately is needed.

## Operating characteristics
- **Time Complexity:** $O(N \cdot C)$, where $N$ is the number of active coordinates and $C$ is the number of channels. The operation involves a single pass over the input coordinates [S1, S2, unofficial].
- **Memory Complexity:** $O(N \cdot C)$ for the output features. The global feature vector is $O(C)$ [S1, S2, unofficial].
- **Strengths:** Efficiently applies global context to sparse data without increasing the dimensionality or changing the coordinate structure [S1, S2, unofficial].
- **Limitations:** Requires the global features to be pre-computed and reduced to the origin [S1, S2, unofficial].
- **When to Use:** When global context needs to modulate local features multiplicatively, such as in attention mechanisms or global-local feature fusion [S1, S2, unofficial].
- **When Not to Use:** When additive or concatenative fusion is more appropriate [S1, S2, unofficial].

## Background
MinkowskiBroadcastMultiplication is part of the Minkowski Engine library, which provides efficient operations for sparse tensors in 3D and higher dimensions [S1, S2, unofficial]. It was introduced in the context of 4D Spatio-Temporal ConvNets, as described in "4D Spatio-Temporal ConvNets: Minkowski Convolutional Neural Networks" (CVPR'19) [S2, unofficial]. The operation is typically used in conjunction with global pooling layers to incorporate global scene information into local sparse feature representations [S1, S2, unofficial].

## Sources
[S1] VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
[S2] VoLoDSplat, https://github.com/pulangk97/VoLoDSplat, fetched 2026-10-27 (unofficial)
