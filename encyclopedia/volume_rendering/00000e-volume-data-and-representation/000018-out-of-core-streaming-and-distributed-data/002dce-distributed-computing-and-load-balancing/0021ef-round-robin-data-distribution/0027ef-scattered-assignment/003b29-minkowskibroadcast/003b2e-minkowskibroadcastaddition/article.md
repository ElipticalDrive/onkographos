# MinkowskiBroadcastAddition

Also called: BroadcastAddition, MinkowskiBroadcastBase with BroadcastMode.ELEMENTWISE_ADDITON

## Overview
MinkowskiBroadcastAddition broadcasts reduced global features to all input coordinates by adding them element-wise to the input sparse tensor features [S1, S2, unofficial]. It solves the problem of injecting global context (e.g., from global pooling) into a sparse tensor's local features while preserving the original coordinate structure [S1, S2, unofficial]. It sets itself apart from [MinkowskiBroadcastMultiplication](8929-minkowskibroadcastmultiplication.md) by using addition instead of multiplication, and from [MinkowskiBroadcastConcatenation](8933-minkowskibroadcastconcatenation.md) by maintaining the original feature dimensionality rather than expanding it [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A `SparseTensor` with features $\mathbf{x}_1$ and a coordinate map key. The features $\mathbf{x}_1$ define the output coordinates [S1, S2, unofficial].
- **Input Glob:** A `SparseTensor` with features $\mathbf{x}_2$ reduced to the origin (e.g., via global pooling) [S1, S2, unofficial].
- **Output:** A `SparseTensor` with the same coordinate map key as the input, containing the broadcasted features [S1, S2, unofficial].

**Data Structures:**
- **SparseTensor:** Contains features (tensor) and coordinate information (coordinate map key, coordinate manager) [S1, S2, unofficial].
- **Coordinate Map Key:** Identifies the coordinate system and resolution [S1, S2, unofficial].
- **Coordinate Manager:** Manages the mapping between batch indices and coordinate rows [S1, S2, unofficial].

**Parameters:**
- **operation_type:** Set to `BroadcastMode.ELEMENTWISE_ADDITON` [S2, unofficial].

**Steps:**
1. Initialize the output feature tensor with the same shape as the input features $\mathbf{x}_1$ [S1, S2, unofficial].
2. Retrieve the batch indices and corresponding row indices for the input coordinates from the coordinate manager's origin map using the input's coordinate map key [S1, S2, unofficial].
3. For each batch index $b$ and its associated rows:
   - Extract the global feature vector $\mathbf{x}_2$ for batch $b$ from the `input_glob` sparse tensor [S1, S2, unofficial].
   - Add $\mathbf{x}_2$ element-wise to the input features $\mathbf{x}_{1, \mathbf{u}}$ for all coordinates $\mathbf{u}$ in the current batch [S1, S2, unofficial].
4. Construct and return a new `SparseTensor` with the computed features, the original coordinate map key, and the original coordinate manager [S1, S2, unofficial].

**Formula:**
$$
\mathbf{y}_\mathbf{u} = \mathbf{x}_{1, \mathbf{u}} + \mathbf{x}_2 \; \text{for} \; \mathbf{u} \in \mathcal{C}^\text{in}
$$
Where:
- $\mathbf{y}_\mathbf{u}$ is the output feature vector at coordinate $\mathbf{u}$ [S1, S2, unofficial].
- $\mathbf{x}_{1, \mathbf{u}}$ is the input feature vector at coordinate $\mathbf{u}$ [S1, S2, unofficial].
- $\mathbf{x}_2$ is the global feature vector reduced to the origin [S1, S2, unofficial].
- $\mathcal{C}^\text{in}$ is the set of input coordinates [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def MinkowskiBroadcastAddition(input: SparseTensor, input_glob: SparseTensor) -> SparseTensor:
    # Initialize output features with the same shape as input features
    broadcast_feat = input.F.new(len(input), input_glob.size()[1])
    
    # Get batch indices and row indices for the input coordinates
    batch_indices, batch_rows = input.coordinate_manager.origin_map(input.coordinate_map_key)
    
    # Iterate over each batch and its corresponding rows
    for b, rows in zip(batch_indices, batch_rows):
        # Extract global feature vector for batch b
        global_feat = input_glob.F[b]
        
        # Add global feature to input features for all coordinates in this batch
        broadcast_feat[rows] = input.F[rows] + global_feat
        
    # Return new SparseTensor with broadcasted features
    return SparseTensor(
        broadcast_feat,
        coordinate_map_key=input.coordinate_map_key,
        coordinate_manager=input.coordinate_manager,
    )
```

**Edge Cases and Pitfalls:**
- **Precision:** Ensure that the addition operation does not suffer from numerical instability, especially if feature values are very large or small [G].
- **Boundaries:** The output coordinates are identical to the input coordinates, so no boundary checks are needed for coordinate validity [S1, S2, unofficial].
- **Degenerate Input:** If `input_glob` is empty or has incorrect dimensions, the operation will fail [G].

## Subtopics

### [MinkowskiBroadcastMultiplication](8929-minkowskibroadcastmultiplication.md)
Uses element-wise multiplication instead of addition. Choose when multiplicative interaction is desired.

### [MinkowskiBroadcastConcatenation](8933-minkowskibroadcastconcatenation.md)
Concatenates global features to input features, increasing dimensionality. Choose when preserving both local and global features explicitly is needed.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of input coordinates, as each coordinate's features are updated once [S1, S2, unofficial].
- **Memory Complexity:** Linear with respect to the number of input coordinates, as a new feature tensor of the same size is created [S1, S2, unofficial].
- **Strengths:** Simple and efficient way to inject global context; preserves coordinate structure [S1, S2, unofficial].
- **Limitations:** Only supports element-wise addition; does not expand feature dimensions [S1, S2, unofficial].
- **When to Use:** When global context needs to be added to local features without changing dimensionality [S1, S2, unofficial].
- **When Not to Use:** When multiplicative interaction or feature expansion is required [S1, S2, unofficial].

## Background
MinkowskiBroadcastAddition is part of the Minkowski Engine library, which is designed for efficient processing of sparse tensors in 3D and higher-dimensional spaces [S1, S2, unofficial]. It was introduced in the context of 4D Spatio-Temporal ConvNets, as described in "4D Spatio-Temporal ConvNets: Minkowski Convolutional Neural Networks" (CVPR'19) [S2, unofficial]. The method is commonly used in applications involving sparse 3D data, such as point cloud processing and volumetric rendering [S1, S2, unofficial].

## Sources
[S1] VolSplat, https://github.com/ziplab/VolSplat, fetched 2026-10-27 (unofficial)
[S2] VoLoDSplat, https://github.com/pulangk97/VoLoDSplat, fetched 2026-10-27 (unofficial)
