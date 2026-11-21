# MinkowskiBroadcast

Also called: BroadcastCopy, Copy Broadcast

## Overview
MinkowskiBroadcast injects global context into a sparse tensor by copying a reduced global feature vector to every active coordinate. It solves the problem of aligning a single global summary (e.g., from global pooling) with local sparse features while preserving the original coordinate structure [S1, S2, unofficial]. It sets itself apart from [MinkowskiBroadcastAddition](8928-minkowskibroadcastaddition.md) and [MinkowskiBroadcastMultiplication](8929-minkowskibroadcastmultiplication.md) by performing a direct copy operation rather than element-wise arithmetic, and from [MinkowskiBroadcastConcatenation](8933-minkowskibroadcastconcatenation.md) by not expanding the feature dimensionality [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** `input`, a `SparseTensor` defining the output coordinates and containing local features.
*   **Input:** `input_glob`, a `SparseTensor` containing features reduced to the origin (global context).
*   **Output:** A new `SparseTensor` with the same coordinates as `input`, but with features replaced by the broadcasted global features.

**Data Structures:**
*   `SparseTensor`: Contains features `F` (tensor of shape `[N, C]` where `N` is number of active coordinates and `C` is feature dimension) and a `coordinate_manager` which maps coordinates to memory locations.
*   `coordinate_manager.origin_map(key)`: Returns a tuple `(batch_indices, batch_rows)` where `batch_indices` is a list of batch IDs and `batch_rows` is a list of index arrays corresponding to the rows in the feature tensor for each batch.

**Steps:**
1.  Assert that both `input` and `input_glob` are instances of `SparseTensor` [S1, S2, unofficial].
2.  Allocate a new feature tensor `broadcast_feat` with shape `(len(input), input_glob.size()[1])` [S1, S2, unofficial]. The first dimension matches the number of active coordinates in `input`, and the second dimension matches the feature dimension of `input_glob`.
3.  Retrieve the mapping from the coordinate manager: `batch_indices, batch_rows = input.coordinate_manager.origin_map(input.coordinate_map_key)` [S1, S2, unofficial].
4.  Iterate over each batch index `b` and its corresponding row indices `rows` from the zip of `batch_indices` and `batch_rows` [S1, S2, unofficial].
5.  For each batch, copy the global feature vector `input_glob.F[b]` to the corresponding rows in `broadcast_feat`: `broadcast_feat[rows] = input_glob.F[b]` [S1, S2, unofficial].
6.  Return a new `SparseTensor` initialized with `broadcast_feat`, `input.coordinate_map_key`, and `input.coordinate_manager` [S1, S2, unofficial].

**Formula:**
The output feature $\mathbf{y}_\mathbf{u}$ for each coordinate $\mathbf{u}$ in the input coordinate set $\mathcal{C}^\text{in}$ is defined as:
$$ \mathbf{y}_\mathbf{u} = \mathbf{x}_2 $$
where $\mathbf{x}_2$ is the global feature vector from `input_glob` corresponding to the batch of $\mathbf{u}$ [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def MinkowskiBroadcast(input: SparseTensor, input_glob: SparseTensor) -> SparseTensor:
    # Assert inputs are SparseTensors
    assert isinstance(input, SparseTensor)
    assert isinstance(input_glob, SparseTensor)

    # Allocate broadcast_feat tensor of size (len(input), input_glob.size()[1])
    broadcast_feat = input.F.new(len(input), input_glob.size()[1])

    # Get batch indices and rows corresponding to the origin for the input coordinate map key
    batch_indices, batch_rows = input.coordinate_manager.origin_map(input.coordinate_map_key)

    # Iterate through batch indices and rows, assigning the global feature for batch b to the broadcast_feat rows
    for b, row_ind in zip(batch_indices, batch_rows):
        broadcast_feat[row_ind] = input_glob.F[b]

    # Return a SparseTensor with broadcast_feat and the input coordinate map key
    return SparseTensor(
        broadcast_feat,
        coordinate_map_key=input.coordinate_map_key,
        coordinate_manager=input.coordinate_manager
    )
```

**Parameters:**
*   `input`: SparseTensor defining the output coordinates.
*   `input_glob`: SparseTensor containing features reduced to the origin [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Batch Mismatch:** The method assumes `input_glob` has one feature vector per batch present in `input`. If `input_glob` is missing a batch index present in `input`, the copy operation will fail or produce undefined behavior [G].
*   **Dimension Mismatch:** `input_glob.size()[1]` (feature dimension) must match the expected output feature dimension. The output tensor `broadcast_feat` is created with this dimension, so if `input_glob` has a different feature size than intended, the output will be malformed [G].

## Subtopics

### [MinkowskiBroadcastAddition](8928-minkowskibroadcastaddition.md)
Broadcasts reduced global features to all input coordinates by adding them element-wise to the input sparse tensor features. It solves the problem of injecting global context while preserving the original coordinate structure. It sets itself apart from [MinkowskiBroadcastMultiplication](8929-minkowskibroadcastmultiplication.md) by using addition instead of multiplication, and from [MinkowskiBroadcastConcatenation](8933-minkowskibroadcastconcatenation.md) by maintaining the original feature dimensionality rather than expanding it.

### [MinkowskiBroadcastMultiplication](8929-minkowskibroadcastmultiplication.md)
Broadcasts a reduced global feature vector to all active coordinates of a sparse tensor by multiplying it element-wise with the input features. It solves the problem of applying global context to local sparse features without changing the coordinate structure. It differs from [MinkowskiBroadcastAddition](8928-minkowskibroadcastaddition.md) and [MinkowskiBroadcastConcatenation](8933-minkowskibroadcastconcatenation.md) by performing element-wise multiplication.

### [MinkowskiBroadcastConcatenation](8933-minkowskibroadcastconcatenation.md)
Broadcasts reduced global features to all input coordinates and concatenates them to the input features along the feature dimension. It solves the problem of injecting global context into sparse tensor representations by aligning a single global feature vector with every active coordinate. Unlike sibling methods that perform element-wise arithmetic operations, this method preserves both the local and global information as separate channels. It builds on [MinkowskiBroadcast](8927-minkowskibroadcast.md).

## Operating characteristics
**Time Complexity:** $O(N)$, where $N$ is the number of active coordinates in `input`. The operation involves iterating through the coordinates once to copy the global feature [S1, S2, unofficial].
**Memory Complexity:** $O(N \cdot C)$, where $C$ is the feature dimension of `input_glob`. A new feature tensor of size $N \times C$ is allocated [S1, S2, unofficial].
**Strengths:** Simple and efficient injection of global context. Preserves coordinate structure exactly.
**Limitations:** Does not allow the network to learn how to combine local and global features (unlike concatenation or multiplication). Requires `input_glob` to be pre-reduced (e.g., via global pooling) [S1, S2, unofficial].
**When to use:** When global context is needed but the specific interaction with local features is handled by subsequent layers or when a direct copy is sufficient for the task [S1, S2, unofficial].

## Background
MinkowskiBroadcast is part of the MinkowskiEngine library, which implements sparse convolutional neural networks. It is designed to work with sparse tensors, which are efficient representations of high-dimensional data with many zero values. The method is typically used in conjunction with global pooling operations to inject global information into sparse feature maps [S1, S2, unofficial]. The underlying sparse tensor representation and coordinate management are based on the Minkowski Convolutional Neural Networks framework [S2, unofficial].

## Sources
[S1] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
[S2] pulangk97/VoLoDSplat, https://github.com/pulangk97/VoLoDSplat, fetched 2026-10-27 (unofficial)
