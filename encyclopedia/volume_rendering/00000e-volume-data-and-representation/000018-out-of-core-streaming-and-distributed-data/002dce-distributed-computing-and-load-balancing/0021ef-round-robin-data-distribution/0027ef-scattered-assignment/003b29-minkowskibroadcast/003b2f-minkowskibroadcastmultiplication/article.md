# MinkowskiBroadcastMultiplication

Also called: BroadcastMultiplication, MinkowskiBroadcastBase with BroadcastMode.ELEMENTWISE_MULTIPLICATION

## Overview
MinkowskiBroadcastMultiplication broadcasts a reduced global feature vector to all active coordinates of a sparse tensor by multiplying it element-wise with the input features [S1, unofficial][S2, unofficial]. It solves the problem of applying global context (e.g., from global pooling) to local sparse features without changing the coordinate structure [S1, unofficial][S2, unofficial]. It builds on [MinkowskiBroadcast](8927-minkowskibroadcast.md) and differs from [MinkowskiBroadcastAddition](8928-minkowskibroadcastaddition.md) (which adds the global vector) and [MinkowskiBroadcastConcatenation](8933-minkowskibroadcastconcatenation.md) (which concatenates the global vector) by performing element-wise multiplication [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A `SparseTensor` with features $\mathbf{x}_1$ and a coordinate map key [S2, unofficial].
- **Input Glob:** A `SparseTensor` with features $\mathbf{x}_2$ reduced to the origin [S2, unofficial].
- **Output:** A `SparseTensor` with the same coordinate map key as the input, containing the multiplied features [S1, unofficial][S2, unofficial].

**Data Structures:**
- `SparseTensor`: Contains a feature tensor and a `coordinate_map_key` [S1, unofficial][S2, unofficial].
- `CoordinateManager`: Manages coordinate maps and batch indices [S1, unofficial][S2, unofficial].
- `BroadcastMode`: Enum constant `ELEMENTWISE_MULTIPLICATION` [S2, unofficial].

**Numbered Steps:**
1. Initialize the operation with `BroadcastMode.ELEMENTWISE_MULTIPLICATION` [S2, unofficial].
2. Retrieve the batch indices and row offsets for the input coordinates from the `coordinate_manager` using the input's `coordinate_map_key` [S1, unofficial][S2, unofficial].
3. Allocate an output feature tensor of the same size as the input features [G].
4. For each batch index $b$ and corresponding row offsets `rows`:
   - Extract the global feature vector $\mathbf{x}_2$ for batch $b$ from `input_glob.F` [S1, unofficial][S2, unofficial].
   - Assign $\mathbf{x}_2$ to all positions in the output feature tensor corresponding to `rows` [S1, unofficial][S2, unofficial].
5. Multiply the assigned global features element-wise with the input features $\mathbf{x}_1$ at those positions [S2, unofficial].
6. Return a new `SparseTensor` with the computed features and the input's `coordinate_map_key` [S1, unofficial][S2, unofficial].

**Formula:**
$$ \mathbf{y}_\mathbf{u} = \mathbf{x}_{1, \mathbf{u}} \times \mathbf{x}_2 \quad \text{for} \quad \mathbf{u} \in \mathcal{C}^\text{in} $$
Where:
- $\mathbf{y}_\mathbf{u}$ is the output feature vector at coordinate $\mathbf{u}$ [S1, unofficial][S2, unofficial].
- $\mathbf{x}_{1, \mathbf{u}}$ is the input feature vector at coordinate $\mathbf{u}$ [S1, unofficial][S2, unofficial].
- $\mathbf{x}_2$ is the global feature vector (reduced to the origin) [S1, unofficial][S2, unofficial].
- $\times$ denotes element-wise multiplication [S2, unofficial].
- $\mathcal{C}^\text{in}$ is the set of input coordinates [S1, unofficial][S2, unofficial].

**Parameters:**
- `operation_type`: Set to `BroadcastMode.ELEMENTWISE_MULTIPLICATION` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def minkowski_broadcast_multiplication(input_features, input_glob_features, batch_indices, batch_rows, coordinate_map_key):
    """
    Broadcasts reduced global features to all input coordinates by multiplying them element-wise.
    
    Args:
        input_features: Array of shape [N, C] containing x1.
        input_glob_features: Array of shape [B, C] containing x2 reduced to origin.
        batch_indices: List of batch IDs.
        batch_rows: List of arrays, each containing row indices for the corresponding batch.
        coordinate_map_key: Key for the output coordinate map.
        
    Returns:
        Tuple of (output_features, coordinate_map_key).
    """
    # Allocate output feature tensor of the same size as input features [G]
    output_features = input_features.copy()
    
    # Iterate over batches
    for b, rows in zip(batch_indices, batch_rows):
        # Extract the global feature vector x2 for batch b [S1, unofficial][S2, unofficial]
        global_feat = input_glob_features[b]
        
        # Assign x2 to all positions corresponding to rows, then multiply element-wise with x1 [S2, unofficial]
        output_features[rows] = input_features[rows] * global_feat
        
    # Return output features and input coordinate map key [S1, unofficial][S2, unofficial]
    return output_features, coordinate_map_key
```

**Edge Cases and Pitfalls:**
- **Dimension Mismatch:** The number of channels in `input_glob_features` must match the number of channels in `input_features` [G].
- **Empty Input:** If the input sparse tensor has no active coordinates, the output should also be empty [G].
- **Batch Indexing:** Ensure that `batch_indices` and `batch_rows` correctly map to the global features, which are typically reduced to the origin per batch [S1, unofficial][S2, unofficial].

## Subtopics

### [MinkowskiBroadcastAddition](8928-minkowskibroadcastaddition.md)
Adds the global feature vector to the input features. Choose when additive context is desired.

### [MinkowskiBroadcastConcatenation](8933-minkowskibroadcastconcatenation.md)
Concatenates the global feature vector to the input features. Choose when preserving both local and global features separately is needed.

## Operating characteristics
- **Time Complexity:** $O(N \cdot C)$, where $N$ is the number of active coordinates and $C$ is the number of channels [G].
- **Memory Complexity:** $O(N \cdot C)$ for the output features [G].
- **Strengths:** Efficiently applies global context to sparse data without increasing the dimensionality or changing the coordinate structure [S1, unofficial][S2, unofficial].
- **Limitations:** Requires the global features to be pre-computed and reduced to the origin [S1, unofficial][S2, unofficial].
- **When to Use:** When global context needs to modulate local features multiplicatively [S1, unofficial][S2, unofficial].

## Background
The operation is documented in the VolSplat and VoLoDSplat repositories [S1, unofficial][S2, unofficial]. It is typically used in conjunction with global pooling layers to incorporate global scene information into local sparse feature representations [S1, unofficial][S2, unofficial].

## Sources
[S1] VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
[S2] VoLoDSplat, https://github.com/pulangk97/VoLoDSplat, fetched 2026-10-27 (unofficial)
