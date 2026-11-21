# Voxel Index Computation

Also called: ComputeIndex3, Index Computation (3-bit)

## Overview
Voxel Index Computation maps continuous 3D world coordinates to discrete integer indices within a voxel grid, enabling efficient data lookup in volume rendering and neural rendering pipelines. It differs from its siblings by supporting two distinct modes: batch-aware mapping for Gaussian splatting applications ([Uniform Grid Indexing](6941-uniform-grid-indexing.md)) and compact 3-bit texel indexing for texture compression ([Uniform Grid Adjacent Cell Indexing](2992-uniform-grid-adjacent-cell-indexing.md)).

## How it works
The method operates in two distinct variants depending on the application context: Batch Voxel Indexing for point cloud/Gaussian splatting queries, and 3-bit Texel Indexing for block compression.

### Inputs and Outputs
**Variant 1: Batch Voxel Indexing**
- **Inputs**:
  - `points`: Tensor of shape $(N, 3)$ containing 3D coordinates.
  - `batch_ids`: Tensor of shape $(N,)$ or $(1,)$ identifying the bounding box for each point.
  - `bounding_boxes`: Tensor of shape $(B, 2, 3)$ where $B$ is the number of batches. `bounding_boxes[b, 0]` is the minimum corner, `bounding_boxes[b, 1]` is the maximum corner.
  - `voxel_size`: Scalar or Tensor of shape $(B,)$ or $(B, 3)$ defining the voxel dimensions.
  - `shift`: Boolean flag. If true, centers the voxel grid relative to the bounding box minimum.
  - `cat_batch_ids`: Boolean flag. If true, appends the batch ID to the output.
- **Outputs**:
  - `voxel_indices`: Tensor of shape $(N, 3)$ if `cat_batch_ids` is false, or $(N, 4)$ if true.

**Variant 2: 3-bit Texel Indexing**
- **Inputs**:
  - `texelPos`: Continuous position within a block.
  - `end0`: Start position of the block.
  - `end1`: End position of the block.
- **Outputs**:
  - `index`: Integer in range $[0, 7]$.

### Data Structures
- **Bounding Box**: Axis-aligned, defined by min and max vectors.
- **Voxel Size**: Can be isotropic (scalar) or anisotropic (vector).
- **Indices**: Signed or unsigned integers depending on implementation; typically 32-bit integers for batch indexing, 3-bit unsigned for texel indexing.

### Steps

#### Variant 1: Batch Voxel Indexing
1.  **Select Bounding Box Minimums**: Retrieve the minimum corner for each point's batch ID.
    $$ \mathbf{b}_{min} = \text{bounding\_boxes}[\text{batch\_ids}][:, 0] $$
2.  **Retrieve Voxel Size**: Get the voxel size for each point. If `voxel_size` is a scalar, it is broadcast. If it is a tensor, it is indexed by `batch_ids`.
    $$ \mathbf{s} = \text{voxel\_size}[\text{batch\_ids}] \quad \text{if tensor, else scalar} $$
3.  **Expand Dimensions**: If $\mathbf{s}$ is 1D (shape $(N,)$ or $(B,)$), expand it to 2D (shape $(N, 1)$ or $(B, 1)$) to allow broadcasting with 3D coordinates.
    $$ \mathbf{s}_{expanded} = \text{unsqueeze}(\mathbf{s}, \text{dim}=-1) $$
4.  **Apply Shift (Optional)**: If the `shift` flag is true, adjust the bounding box minimums to center the first voxel.
    $$ \mathbf{b}_{min}' = \mathbf{b}_{min} - 0.5 \times \mathbf{s}_{expanded} $$
    Otherwise, $\mathbf{b}_{min}' = \mathbf{b}_{min}$.
5.  **Compute Indices**: Calculate the floating-point coordinates relative to the adjusted minimum, divide by voxel size, and floor the result.
    $$ \mathbf{c}_{float} = \frac{\text{points} - \mathbf{b}_{min}'}{\mathbf{s}_{expanded}} $$
    $$ \mathbf{i} = \lfloor \mathbf{c}_{float} \rfloor $$
    Cast $\mathbf{i}$ to long integers.
6.  **Concatenate Batch IDs (Optional)**: If `cat_batch_ids` is true, concatenate `batch_ids` to $\mathbf{i}$ along dimension 1.
    $$ \text{output} = \text{concat}([\mathbf{i}, \text{batch\_ids}], \text{dim}=1) $$
    Otherwise, return $\mathbf{i}$.

#### Variant 2: 3-bit Texel Indexing
1.  **Normalize Position**: Compute the relative position of `texelPos` within the block defined by `end0` and `end1`.
    $$ \text{ratio} = \frac{\text{texelPos} - \text{end0}}{\text{end1} - \text{end0}} $$
2.  **Scale and Offset**: Apply specific scaling and offset constants to map the ratio to the 0-7 range.
    $$ \text{val} = \text{ratio} \times 6.98182 + 0.00909 + 0.5 $$
3.  **Round and Clamp**: Round to the nearest integer and clamp to $[0, 7]$.
    $$ \text{index} = \text{clamp}(\text{round}(\text{val}), 0, 7) $$

### Formulas
**Batch Voxel Indexing**:
$$ i_x = \left\lfloor \frac{p_x - b_{min,x} - \delta_x}{s_x} \right\rfloor $$
where $\delta_x = 0.5 s_x$ if `shift` is true, else $0$.

**3-bit Texel Indexing**:
$$ I = \text{clamp}\left( \text{round}\left( \frac{P - E_0}{E_1 - E_0} \times 6.98182 + 0.50909 \right), 0, 7 \right) $$

### Parameters
- **shift**: Boolean. Default false.
- **cat_batch_ids**: Boolean. Default false.
- **Constants for 3-bit**: $6.98182$ (scale), $0.00909$ (offset).

### Edge Cases and Pitfalls
- **Out-of-Bounds**: Points outside the bounding box will produce negative indices or indices $\ge N$. The source [S1, unofficial] does not explicitly show clamping to $[0, N-1]$ in the core formula, relying on the bounding box definition. Implementers should clamp indices to valid grid bounds to prevent memory access violations [G].
- **Division by Zero**: If `end1 == end0` in Variant 2, division by zero occurs. Clamp or handle this case explicitly [G].
- **Precision**: Floating-point errors in `floor` operations can cause off-by-one errors at exact boundaries. Using `floor` on slightly adjusted values or epsilon checks may be necessary [G].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_voxel_indices_batch(points, batch_ids, bounding_boxes, voxel_size, shift=False, cat_batch_ids=False):
    """
    Variant 1: Batch Voxel Indexing
    points: (N, 3)
    batch_ids: (N,)
    bounding_boxes: (B, 2, 3)
    voxel_size: scalar or (B,) or (B, 3)
    """
    # Step 1: Select min bounding box
    # Assuming batch_ids are indices into the first dimension of bounding_boxes
    bb_mins = bounding_boxes[batch_ids][:, 0]  # (N, 3)
    
    # Step 2: Retrieve voxel size
    if isinstance(voxel_size, (int, float)):
        voxel_size_b = voxel_size
    else:
        voxel_size_b = voxel_size[batch_ids]  # (N,) or (N, 3)
        
    # Step 3: Expand dimensions if 1D
    if hasattr(voxel_size_b, 'dim') and voxel_size_b.dim() == 1:
        voxel_size_b = voxel_size_b.unsqueeze(-1)  # (N, 1)
        
    # Step 4: Apply shift
    if shift:
        bb_mins = bb_mins - 0.5 * voxel_size_b
        
    # Step 5: Compute indices
    # Ensure broadcasting works: points (N,3), bb_mins (N,3), voxel_size_b (N,1) or (N,3)
    voxel_indices = torch.floor((points - bb_mins) / voxel_size_b).long()
    
    # Step 6: Concatenate batch IDs
    if cat_batch_ids:
        # Ensure batch_ids is 2D for concatenation
        batch_ids_2d = batch_ids.unsqueeze(-1).long()
        output = torch.cat([voxel_indices, batch_ids_2d], dim=1)
        return output
    else:
        return voxel_indices

def compute_texel_index_3bit(texelPos, end0, end1):
    """
    Variant 2: 3-bit Texel Indexing
    Returns integer 0-7
    """
    # Avoid division by zero
    if end1 == end0:
        return 0
        
    ratio = (texelPos - end0) / (end1 - end0)
    val = ratio * 6.98182 + 0.00909 + 0.5
    index = round(val)
    # Clamp to 0-7
    if index < 0:
        index = 0
    elif index > 7:
        index = 7
    return index
```

## Subtopics

### [Lighting Fast Querier](11265-lighting-fast-querier.md)
A CUDA-accelerated module for neural radiance fields that integrates ray generation, jittered sampling, and specific kernel-based query logic optimized for point cloud density. It differs from generic voxel indexing by handling super-resolution scaling and training-specific jitter.

## Operating characteristics
- **Time Complexity**: $O(N)$ for batch indexing, where $N$ is the number of points. The 3-bit indexing is $O(1)$.
- **Memory Complexity**: $O(N)$ to store the output indices.
- **Performance**: Batch indexing is highly parallelizable on GPUs. The 3-bit method is extremely fast but limited to small blocks (2x2x2 texels).
- **Strengths**: Simple arithmetic, GPU-friendly, supports batched queries.
- **Limitations**: Does not handle non-axis-aligned grids. Boundary handling requires explicit clamping in implementation.

## Background
The batch voxel indexing method is used in 2D/3D Gaussian Splatting frameworks, specifically in the VGVGGT project [S1, unofficial]. The 3-bit indexing method originates from real-time BC6H compression on GPU, used for texture block indexing [S2, unofficial].

## Sources
[S1] Zhaoyibinn/VGVGGT, https://github.com/Zhaoyibinn/VGVGGT, fetched 2026-10-27 (unofficial)
[S2] Real-time BC6H compression on GPU, https://github.com/knarkowicz/GPURealTimeBC6H, fetched 2026-10-27 (unofficial)
