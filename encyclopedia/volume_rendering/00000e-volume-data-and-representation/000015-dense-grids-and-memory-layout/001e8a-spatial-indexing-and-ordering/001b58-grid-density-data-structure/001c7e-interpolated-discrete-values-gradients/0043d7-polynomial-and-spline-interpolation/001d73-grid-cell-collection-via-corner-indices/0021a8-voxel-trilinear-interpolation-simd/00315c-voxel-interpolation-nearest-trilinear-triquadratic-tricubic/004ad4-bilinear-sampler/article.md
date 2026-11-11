# Bilinear Sampler


## Overview
The Bilinear Sampler is a differentiable operation that samples values from a dense grid (tensor) at arbitrary floating-point coordinates using bilinear interpolation. It solves the problem of extracting feature maps or voxel data at non-integer locations, which is essential for spatial transformer networks, deformable convolutions, and neural rendering pipelines. It distinguishes itself from nearest-neighbor sampling by providing smooth gradients and from higher-order splines by its computational efficiency and local support over a single cell. It builds on [Voxel Interpolation (Nearest, Trilinear, Triquadratic, Tricubic)](6486-voxel-interpolation-nearest-trilinear-triquadratic-tricubic.md) but is specialized for 2D spatial grids (and 3D spatiotemporal grids) with specific coordinate normalization conventions (`align_corners`) and padding modes.

## How it works
The sampler takes an input tensor and a grid of coordinates, normalizes the coordinates to the range $[-1, 1]$, computes the four (2D) or eight (3D) corner indices, performs weighted interpolation, and handles boundary conditions.

### Inputs and Outputs
- **Input**: A tensor $I$ of shape $(B, C, H, W)$ for 2D or $(B, C, T, H, W)$ for 3D. $B$ is batch size, $C$ is channels, $H, W$ are height/width, $T$ is depth/time.
- **Coordinates**: A tensor $G$ of shape $(B, H_o, W_o, 2)$ for 2D or $(B, T_o, H_o, W_o, 3)$ for 3D. The last dimension contains normalized coordinates.
- **Output**: A tensor $O$ of shape $(B, C, H_o, W_o)$ for 2D or $(B, C, T_o, H_o, W_o)$ for 3D.

### Data Structures
- **Input Tensor**: Standard N-dimensional tensor.
- **Grid Tensor**: Stores normalized coordinates. For 2D, each point is $(x, y)$. For 3D, each point is $(x, y, t)$ or $(t, x, y)$ depending on implementation details (see steps).

### Parameters
- **align_corners** (bool): If `True`, the coordinate range $[-1, 1]$ maps to the pixel centers at indices $0$ and $H-1$ (or $W-1$). If `False`, the range maps to the pixel edges, effectively treating pixels as uniform bins. Default behavior varies by library; PyTorch defaults to `False` for recent versions but `True` for older ones.
- **padding_mode** (str): How to handle coordinates outside $[-1, 1]$. Options: `'zeros'`, `'border'`, `'reflection'`. Default is often `'border'` or `'zeros'`.

### Steps
1. **Preprocessing**: Clone and detach the coordinate tensor $G$. Convert $G$ to the same device and data type as the input tensor $I$.
2. **Dimension Check**: Assert that the spatial dimensions of $I$ are either 2 or 3.
3. **3D Reordering**: If the input is 3D (shape includes $T, H, W$), reorder the coordinate channels in $G$ from $(t, x, y)$ to $(x, y, t)$ to match the spatial layout expected by the underlying grid sample kernel. Specifically, $G[..., [1, 2, 0]] = G$.
4. **Normalization**:
   - Let $S$ be the list of spatial sizes of $I$ (e.g., $[H, W]$ or $[T, H, W]$).
   - Compute the scale factor for each dimension $s \in S$:
     - If `align_corners` is `True`: $scale = \frac{2}{\max(s - 1, 1)}$.
     - If `align_corners` is `False`: $scale = \frac{2}{s}$.
   - Normalize coordinates: $G_{norm} = G \cdot scale - 1$.
   - This maps integer grid indices $0 \dots s-1$ to $[-1, 1]$ (align_corners) or $[-1 + \frac{2}{s}, 1 - \frac{2}{s}]$ (not align_corners).
5. **Interpolation**:
   - For each output point $(x, y)$ in $G_{norm}$:
     - Compute grid coordinates: $x_{grid} = \frac{x + 1}{2} \cdot W$, $y_{grid} = \frac{y + 1}{2} \cdot H$.
     - Identify corner indices: $x_0 = \lfloor x_{grid} \rfloor, x_1 = x_0 + 1, y_0 = \lfloor y_{grid} \rfloor, y_1 = y_0 + 1$.
     - Compute weights: $w_x = x_{grid} - x_0, w_y = y_{grid} - y_0$.
     - Fetch values $I_{c, y_0, x_0}, I_{c, y_0, x_1}, I_{c, y_1, x_0}, I_{c, y_1, x_1}$ for each channel $c$.
     - Apply boundary conditions based on `padding_mode` if indices are out of bounds.
     - Interpolate: $O_{c, y, x} = (1-w_x)(1-w_y)I_{y_0, x_0} + w_x(1-w_y)I_{y_0, x_1} + (1-w_x)w_y I_{y_1, x_0} + w_x w_y I_{y_1, x_1}$.
6. **Return**: The output tensor $O$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bilinear_sampler(input_tensor, coords, align_corners=False, padding_mode='border'):
    """
    input_tensor: (B, C, H, W) or (B, C, T, H, W)
    coords: (B, H_o, W_o, 2) or (B, T_o, H_o, W_o, 3)
    """
    # 1. Preprocessing
    coords = coords.clone().detach().to(device=input_tensor.device, dtype=input_tensor.dtype)
    sizes = input_tensor.shape[2:] # H, W or T, H, W
    
    # 2. Dimension Check
    assert len(sizes) in [2, 3], "Input must be 2D or 3D"
    
    # 3. 3D Reordering
    if len(sizes) == 3:
        # Reorder from (t, x, y) to (x, y, t)
        coords = coords[..., [1, 2, 0]]
        
    # 4. Normalization
    normalized_coords = torch.zeros_like(coords)
    for i, size in enumerate(reversed(sizes)):
        if align_corners:
            scale = 2.0 / max(size - 1, 1)
        else:
            scale = 2.0 / size
        # Apply scale and shift
        # Note: coords are in the last dim, so we iterate over the last dim indices
        # The loop above iterates reversed(sizes), so we map to the last 2 or 3 dims of coords
        # This is a simplified representation; actual implementation uses vectorized ops
        normalized_coords[..., i] = coords[..., i] * scale - 1.0
        
    # 5. Interpolation (Conceptual)
    # The actual interpolation is performed by the underlying grid_sample kernel
    # which handles the corner fetching, weighting, and padding.
    output = grid_sample_kernel(input_tensor, normalized_coords, padding_mode)
    
    return output
```

### Edge Cases and Pitfalls
- **Precision**: Floating-point errors in coordinate normalization can cause off-by-one errors in corner index calculation. Use stable floor operations.
- **Boundaries**: Coordinates outside $[-1, 1]$ must be handled. `padding_mode='border'` clamps to the edge values. `padding_mode='zeros'` pads with 0.
- **Align Corners**: Misunderstanding `align_corners` leads to sampling at the wrong physical location. When `align_corners=True`, the corners of the grid are aligned with the corners of the image. When `False`, the centers of the corner pixels are at $\pm 1$.

## Subtopics

### Sample Features 4D
A variant that samples spatial features from a 4D tensor $(B, C, H, W)$ at given coordinates $(B, R, 2)$, returning features per point $(B, R, C)$. It wraps the standard bilinear sampler by reshaping the output from $(B, C, R, 1)$ to $(B, R, C)$ via permutation and view operations. It is used in point-based feature extraction where coordinates are sparse.

### PointRend Point Sampling
A specialized sampling strategy used in PointRend for adaptive point sampling. It samples features at specific 2D coordinates using `point_sample`, which normalizes coordinates from $[0, 1]$ to $[-1, 1]$ via $2.0 \cdot p - 1.0$. It supports uncertainty-based sampling where points are selected based on prediction uncertainty, using an oversample ratio and importance sampling ratio to refine the point set. It generates regular grid coordinates and applies affine transformations to align with the feature map.

## Operating characteristics
- **Time Complexity**: $O(N \cdot C)$, where $N$ is the number of sample points and $C$ is the number of channels. Each point requires 4 (2D) or 8 (3D) memory accesses and linear interpolation.
- **Memory Complexity**: $O(N \cdot C)$ for the output. The input tensor is read-only.
- **Performance**: Highly optimized in modern deep learning frameworks (PyTorch, TensorFlow) using SIMD instructions and GPU kernels. Performance is limited by memory bandwidth for large feature maps.
- **Strengths**: Differentiable, fast, simple implementation.
- **Limitations**: Only bilinear accuracy. Not suitable for high-frequency content where higher-order interpolation is needed. Boundary handling can introduce artifacts if not configured correctly.

## Background
The Bilinear Sampler is a fundamental component in Spatial Transformer Networks (STNs) [S1, unofficial] and deformable convolutional networks. It was popularized by the introduction of `F.grid_sample` in PyTorch, which provides a unified interface for sampling. The method is widely used in neural rendering, image warping, and feature alignment tasks. The specific coordinate normalization and padding modes are designed to integrate seamlessly with deep learning frameworks' tensor operations.

## Sources
[S1] InternRobotics/AnySplat, https://github.com/InternRobotics/AnySplat, fetched 2026-10-27 (unofficial)
[S2] sxyu/pixel-nerf, https://github.com/sxyu/pixel-nerf, fetched 2026-10-26 (unofficial)
