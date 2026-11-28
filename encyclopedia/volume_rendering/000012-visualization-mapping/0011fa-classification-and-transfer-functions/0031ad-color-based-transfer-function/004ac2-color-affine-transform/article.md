# Color Affine Transform

Also called: Color Affine Transformation

## Overview
Color Affine Transform applies a linear mapping with an offset to RGB color vectors to adjust appearance, correct bias, or align rendered outputs with target data. It distinguishes itself from simple linear enhancement (like Windowing) by fitting a per-pixel or per-region affine transform ($ax+b$) via least-squares regression, allowing it to compensate for non-zero-mean noise and exposure variations that simple scaling cannot handle [S1, S2, S3, unofficial]. It is computationally lighter than learned appearance encodings but offers more flexibility than fixed colormaps [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: `rgb`, a tensor of RGB color values with shape `(..., 3)`.
- Input: `affine_mats`, a tensor of affine matrices with shape `(..., 3, 4)`.
- Output: `out`, a tensor of transformed RGB colors with shape `(..., 3)`.

**Data Structures:**
The `affine_mats` tensor stores the transformation parameters. For each element in the batch/voxel grid, it contains a $3 \times 4$ matrix. The first three columns represent the linear transformation matrix $M$ ($3 \times 3$), and the fourth column represents the translation vector $T$ ($3 \times 1$) [S3, unofficial].

**Formula:**
The transformation is defined as:
$$ C_{out} = M C_{in} + T $$
Where:
- $C_{in}$ is the input RGB vector ($3 \times 1$).
- $M$ is the $3 \times 3$ matrix formed by the first three columns of `affine_mats`.
- $T$ is the $3 \times 1$ vector formed by the fourth column of `affine_mats`.
- $C_{out}$ is the output RGB vector.

In tensor notation, this is expressed as:
$$ \text{out} = \text{affine\_mats}[:, :, :3] \cdot \text{rgb}^T + \text{affine\_mats}[:, :, 3] $$
(Summed over the last dimension of `rgb` and the corresponding dimension of the matrix) [S1, S2, S3, unofficial].

**Steps:**
1. Extract the linear component $M$ from `affine_mats` by taking the first three columns: $M = \text{affine\_mats}[:, :, :3]$.
2. Extract the translation component $T$ from `affine_mats` by taking the fourth column: $T = \text{affine\_mats}[:, :, 3]$.
3. Perform matrix multiplication between $M$ and the input RGB vector $C_{in}$ (transposed to column vector if necessary).
4. Add the translation vector $T$ to the result.
5. Return the resulting tensor.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def color_affine_transform(rgb, affine_mats):
    """
    Applies an affine transformation to RGB colors.
    
    Args:
        rgb: Tensor of shape (..., 3) containing RGB values.
        affine_mats: Tensor of shape (..., 3, 4) containing affine matrices.
        
    Returns:
        Tensor of shape (..., 3) containing transformed RGB values.
    """
    # Extract the 3x3 linear transformation matrix (first 3 columns)
    M = affine_mats[..., :3]
    
    # Extract the 3x1 translation vector (4th column)
    T = affine_mats[..., 3]
    
    # Matrix multiplication: M @ rgb.T
    # rgb.unsqueeze(-1) makes it (..., 3, 1)
    # matmul result is (..., 3, 1)
    transformed = torch.matmul(M, rgb.unsqueeze(-1))
    
    # Remove the last dimension to get (..., 3)
    transformed = transformed.squeeze(-1)
    
    # Add translation vector
    out = transformed + T
    
    return out
```

**Parameters:**
- `affine_mats`: Typically pre-computed via least-squares regression to minimize the difference between the transformed output and a target color space [S1, S2, unofficial].
- `rgb`: Values are typically in the range $[0, 1]$ [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Shape Mismatch:** The batch dimensions of `rgb` and `affine_mats` must be compatible for broadcasting. If `rgb` is $(N, 3)$ and `affine_mats` is $(N, 3, 4)$, they align directly. If `affine_mats` is $(1, 3, 4)$, it applies the same transform to all $N$ pixels.
- **Precision:** Floating-point precision issues may arise if the affine matrices are derived from high-ISO noise or extreme exposure variations, though the method is designed to handle such non-zero-mean noise better than linear scaling [S1, S2, unofficial].
- **Degenerate Input:** If `rgb` contains NaNs or Inf values, the output will propagate these. Input validation is recommended [G].

## Subtopics

### [Image-dependent affine color transformation](22480-image-dependent-affine-color-transformation.md)
A variant that computes the affine matrices per-pixel or per-Gaussian based on image-dependent features, rather than using a single global or pre-computed set. It is chosen when appearance varies significantly across the scene due to lighting or sensor noise.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of pixels/voxels, as it involves a constant number of arithmetic operations per pixel (matrix-vector multiplication and addition).
- **Memory Complexity:** $O(N)$ to store the `affine_mats` tensor if it is per-pixel, or $O(1)$ if it is a single global matrix.
- **Performance:** Efficient on GPUs due to tensor operations. The implementation in [S3, unofficial] notes a slight performance improvement by avoiding data copies when `grid_idx` contains unique indices.
- **Strengths:** Simple to implement, differentiable (useful for optimization), and effective for color alignment tasks [S1, S2, S3, unofficial].
- **Limitations:** Assumes a linear relationship between input and output colors, which may not hold for all non-linear display characteristics or complex lighting interactions without pre-compensation [S1, S2, unofficial].

## Background
The method is used in 3D reconstruction and rendering pipelines, particularly in conjunction with diffusion models and Gaussian Splatting, to improve the visual quality of rendered outputs by aligning them with ground truth data. It was popularized in projects like DIFIX3D+ and AnySplat [S1, S2, unofficial]. The technique is also referenced in the context of bilateral grid-based color transformations [S3, unofficial].

## Sources
[S1] nv-tlabs/Difix3D, https://github.com/nv-tlabs/Difix3D, fetched 2026-10-26 (unofficial)
[S2] InternRobotics/AnySplat, https://github.com/InternRobotics/AnySplat, fetched 2026-10-27 (unofficial)
[S3] SNU-ARC/GS-Scale, https://github.com/SNU-ARC/GS-Scale, fetched 2026-10-27 (unofficial)
