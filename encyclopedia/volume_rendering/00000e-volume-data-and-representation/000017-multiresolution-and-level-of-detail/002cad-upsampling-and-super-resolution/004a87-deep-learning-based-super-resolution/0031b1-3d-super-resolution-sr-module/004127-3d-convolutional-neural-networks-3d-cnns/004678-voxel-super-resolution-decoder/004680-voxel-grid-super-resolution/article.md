# Voxel-grid Super-Resolution

Also called: Fast grid upsampling

## Overview
Voxel-grid Super-Resolution predicts an upsampled voxel grid of higher resolution from a lower-resolution input grid using a U-Net style encoder-decoder network [S1]. It serves as a fast alternative to querying an implicit Multi-Layer Perceptron (MLP) for higher resolution data, addressing the computational bottleneck associated with dense sampling in neural implicit representations [S1].

## How it works
**Inputs and Outputs:**
- Input: A low-resolution voxel grid $V_{input}$ with dimensions $160^3$ [S1].
- Output: A high-resolution voxel grid $V_{upsample}$ with dimensions $384^3$ [S1].

**Data Structures:**
The input and output are dense 3D grids. Each voxel contains feature vectors. The specific channel dimension $C$ is not explicitly defined for the base super-resolution task in the sources, though the refinement variant uses $C$ channels [S2].

**Method:**
The transformation is defined as:
$$ V_{upsample} = g(V_{input}, \text{resolution}) $$
where $g$ is a U-Net style encoder-decoder network [S1]. The network takes the low-resolution grid and the target resolution as parameters to generate the higher-resolution representation.

**Parameters:**
- Input resolution: $160^3$ [S1].
- Output resolution: $384^3$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_grid_super_resolution(v_input: Tensor, target_res: int) -> Tensor:
    """
    Upsamples a low-resolution voxel grid to a higher resolution using a U-Net.
    
    Args:
        v_input: Tensor of shape (160, 160, 160, C) representing the input voxel grid.
        target_res: Integer, the target resolution dimension (e.g., 384).
        
    Returns:
        Tensor of shape (target_res, target_res, target_res, C) representing the upsampled grid.
    """
    # The function g is a U-Net style encoder-decoder network.
    # Implementation details of g are not provided in the source text.
    # It typically involves:
    # 1. Encoder path: Downsampling with convolutions to capture context.
    # 2. Bottleneck: Deep feature extraction.
    # 3. Decoder path: Upsampling with convolutions to restore resolution.
    # 4. Skip connections: Concatenating encoder features to decoder features.
    
    v_upsample = g(v_input, target_res)
    return v_upsample
```

## Subtopics

### Sparse 3D Decoder
A refinement stage that improves spatial consistency and structural fidelity of a voxel representation by predicting residual features for each voxel using a sparse 3D U-Net [S2]. It is chosen when the base voxel features require correction for fine geometric details and consistency, stabilizing training by learning correction terms rather than relearning entire feature content [S2]. It builds on [Voxel Super-Resolution Decoder](11760-voxel-super-resolution-decoder.md).

**Definition:**
A sparse convolutional 3D U-Net $R$ predicts a residual voxel field $R$ from an input voxel grid $V$ with per-voxel feature vectors [S2].

**Steps:**
1. Input a voxel grid $V$ with per-voxel feature vectors into a sparse convolutional 3D U-Net $R$ [S2].
2. Predict a residual voxel field $R$ where $R \in \mathbb{R}^{V \times C}$ and $R = R(V)$, with $V$ denoting the set of occupied voxels [S2].
3. Obtain refined voxel features $V'$ by adding the residual field to the original features: $V' = V + R$ [S2].

**Layout:**
The refinement network uses hierarchical sparse 3D convolutional blocks, symmetric encoder-decoder stages, and upsampling layers connected by skip connections [S2].

**Pitfalls:**
Relearning entire feature content can destabilize training; the residual formulation encourages learning correction terms (fine geometric detail and consistency cues) to stabilize training and preserve coarse voxel information [S2].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources. However, the method is described as a "fast alternative" to querying an implicit MLP [S1], implying lower latency for high-resolution generation compared to per-point MLP evaluation. The sparse variant maintains efficiency through sparsity [S2].

**Measured Performance:**
Not given in the sources.

**Strengths and Limitations:**
- Strength: Fast grid upsampling compared to implicit MLP querying [S1].
- Strength: The residual formulation in the sparse variant stabilizes training and preserves coarse information [S2].
- Limitation: Requires training a U-Net architecture; specific accuracy metrics are not provided in the sources.

**When to use it:**
Use when high-resolution voxel grids are needed quickly and querying an implicit MLP is too slow [S1]. Use the sparse variant when spatial consistency and structural fidelity need refinement from coarse voxel features [S2].

## Background
The concept of Voxel-grid Super-Resolution was introduced in the context of self-supervised 3D representation learning for Neural Radiance Fields [S1]. It addresses the need for efficient high-resolution grid generation in 3D vision tasks [S1]. The sparse 3D refinement variant was proposed in the context of feed-forward 3D Gaussian Splatting with voxel-aligned prediction [S2].

## Sources
[S1] NeRF-MAE: Masked AutoEncoders for Self-Supervised 3D Representation Learning for Neural Radiance Fields, https://arxiv.org/abs/2404.01300v3, fetched 2026-10-26
[S2] VolSplat: Rethinking Feed-Forward 3D Gaussian Splatting with Voxel-Aligned Prediction, https://doi.org/10.1007/978-3-032-37232-1_1, fetched 2026-10-26
