# MinkowskiBroadcast

Also called: BroadcastCopy, Copy Broadcast

## Overview
MinkowskiBroadcast copies a reduced global feature vector to every active coordinate of a sparse tensor [S2, unofficial]. It preserves the original coordinate structure while distributing the global features across all input locations [S2, unofficial]. It differs from [MinkowskiBroadcastAddition](8928-minkowskibroadcastaddition.md) and [MinkowskiBroadcastMultiplication](8929-minkowskibroadcastmultiplication.md) by performing a direct copy rather than element-wise arithmetic [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** `input`, a sparse tensor defining the output coordinates [S2, unofficial].
*   **Input:** `input_glob`, a sparse tensor containing features reduced to the origin [S2, unofficial].
*   **Output:** A sparse tensor with the same coordinates as `input`, but with features replaced by the broadcasted global features [S2, unofficial].

**Data Structures:**
*   `SparseTensor`: A container holding a feature tensor `F` and coordinate metadata (`coordinate_map_key`, `coordinate_manager`) [S1, S2, unofficial].
*   `coordinate_manager.origin_map(key)`: Returns a tuple `(batch_indices, batch_rows)` mapping each batch ID to the row indices in the feature tensor for that batch [S1, S2, unofficial].

**Steps:**
1.  Assert that both `input` and `input_glob` are instances of `SparseTensor` [S1, S2, unofficial].
2.  Allocate a new feature tensor `broadcast_feat` with shape `(len(input), input_glob.size()[1])` [S1, S2, unofficial].
3.  Retrieve the batch-to-row mapping: `batch_indices, batch_rows = input.coordinate_manager.origin_map(input.coordinate_map_key)` [S1, S2, unofficial].
4.  Iterate over each batch index `b` and its corresponding row indices `rows` from the paired lists [S1, S2, unofficial].
5.  For each batch, copy the global feature vector `input_glob.F[b]` to the corresponding rows in `broadcast_feat` [S1, S2, unofficial].
6.  Return a new `SparseTensor` initialized with `broadcast_feat`, `input.coordinate_map_key`, and `input.coordinate_manager` [S1, S2, unofficial].

**Formula:**
The output feature $\mathbf{y}_\mathbf{u}$ for each coordinate $\mathbf{u}$ in the input coordinate set $\mathcal{C}^\text{in}$ is defined as:
$$ \mathbf{y}_\mathbf{u} = \mathbf{x}_2 \; \text{for} \; \mathbf{u} \in \mathcal{C}^\text{in} $$
where $\mathbf{x}_2$ is the global feature vector from `input_glob` corresponding to the batch containing $\mathbf{u}$ [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class SparseTensor:
    def __init__(self, F, coordinate_map_key, coordinate_manager):
        self.F = F
        self.coordinate_map_key = coordinate_map_key
        self.coordinate_manager = coordinate_manager
    def __len__(self):
        return len(self.F)
    def size(self, dim):
        return self.F.size(dim)

def MinkowskiBroadcast(input: SparseTensor, input_glob: SparseTensor) -> SparseTensor:
    assert isinstance(input, SparseTensor)
    assert isinstance(input_glob, SparseTensor)

    broadcast_feat = input.F.new(len(input), input_glob.size()[1])
    batch_indices, batch_rows = input.coordinate_manager.origin_map(input.coordinate_map_key)

    for b, row_ind in zip(batch_indices, batch_rows):
        broadcast_feat[row_ind] = input_glob.F[b]

    return SparseTensor(
        broadcast_feat,
        coordinate_map_key=input.coordinate_map_key,
        coordinate_manager=input.coordinate_manager
    )
```

**Parameters:**
*   `input`: SparseTensor defining the output coordinates [S2, unofficial].
*   `input_glob`: SparseTensor containing features reduced to the origin [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Batch Mismatch:** The method assumes `input_glob` has one feature vector per batch present in `input` [G].
*   **Dimension Mismatch:** `input_glob.size()[1]` must match the expected output feature dimension [G].

## Subtopics

### [MinkowskiBroadcastAddition](8928-minkowskibroadcastaddition.md)
MinkowskiBroadcastAddition broadcasts reduced global features to all input coordinates by adding them element-wise to the input sparse tensor features. It solves the problem of injecting global context (e.g., from global pooling) into a sparse tensor's local features while preserving the original coordinate structure. It sets itself apart from [MinkowskiBroadcastMultiplication](8929-minkowskibroadcastmultiplication.md) by using addition instead of multiplication, and from [MinkowskiBroadcastConcatenation](8933-minkowskibroadcastconcatenation.md) by maintaining the original feature dimensionality rather than expanding it.

### [MinkowskiBroadcastMultiplication](8929-minkowskibroadcastmultiplication.md)
MinkowskiBroadcastMultiplication broadcasts a reduced global feature vector to all active coordinates of a sparse tensor by multiplying it element-wise with the input features. It solves the problem of applying global context (e.g., from global pooling) to local sparse features without changing the coordinate structure. It differs from [MinkowskiBroadcastAddition](8928-minkowskibroadcastaddition.md) (which adds the global vector) and [MinkowskiBroadcastConcatenation](8933-minkowskibroadcastconcatenation.md) (which concatenates the global vector to the input features) by performing element-wise multiplication.

### [MinkowskiBroadcastConcatenation](8933-minkowskibroadcastconcatenation.md)
MinkowskiBroadcastConcatenation broadcasts reduced global features to all input coordinates and concatenates them to the input features along the feature dimension. It aligns a single global feature vector with every active coordinate in the input tensor. Unlike sibling methods that perform element-wise arithmetic operations between local and global features, this method preserves both information types as separate channels. It builds on [MinkowskiBroadcast](8927-minkowskibroadcast.md) [G].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] ziplab/VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
[S2] pulangk97/VoLoDSplat, https://github.com/pulangk97/VoLoDSplat, fetched 2026-10-27 (unofficial)
