# Bilinear Sampler

Also called: Grid Sample Wrapper, 3D/2D Grid Sampling

## Overview
The Bilinear Sampler is a differentiable operation that samples values from a dense grid (image or volume) at arbitrary floating-point coordinates using bilinear (2D) or trilinear (3D) interpolation [S1, unofficial]. It serves as the bridge between continuous coordinate spaces (such as ray-casting outputs or feature point locations) and discrete tensor representations [S1, unofficial]. It differs from its parent [Voxel Interpolation](6486-voxel-interpolation-nearest-trilinear-triquadratic-tricubic.md) by being a specific, optimized implementation of order-1 interpolation, typically exposed via deep learning frameworks (e.g., PyTorch's `F.grid_sample`) with specific coordinate normalization conventions (`align_corners`) and padding modes [S1, unofficial].

## How it works
The method maps input coordinates from pixel/voxel space to normalized grid space $[-1, 1]$, performs interpolation, and handles boundary conditions.

### Inputs and Outputs
*   **Input Grid ($I$):** A tensor of shape $(B, C, H, W)$ for 2D or $(B, C, T, H, W)$ for 3D, where $B$ is batch size, $C$ is channels, and $H, W, T$ are spatial dimensions [S1, unofficial].
*   **Coordinates ($U$):** A tensor of shape $(B, N, 2)$ for 2D or $(B, N, 3)$ for 3D, representing the sampling locations in pixel/voxel space [S1, unofficial]. For batched feature sampling, $N$ may be replaced by spatial dimensions $(H_o, W_o)$ or $(T_o, H_o, W_o)$ [S1, unofficial].
*   **Output ($O$):** A tensor of shape $(B, C, N)$ or $(B, C, H_o, W_o)$ containing the interpolated values [S1, unofficial].

### Data Structures and Layout
*   **Grid:** Stored in memory as a contiguous tensor. For 3D sampling, the spatial dimensions are often transposed to match the framework's expected layout (e.g., $D, H, W$ instead of $H, W, D$) before sampling [S1, unofficial].
*   **Coordinates:** Stored as floating-point values. The last dimension contains the spatial coordinates $(x, y)$ or $(x, y, z)$ [S1, unofficial].

### Parameters
*   **`align_corners` (bool):** Determines the coordinate mapping.
    *   If `True`: The corners of the grid correspond to pixel centers. The range is $[0, \text{size}-1]$. The normalization scale is $2 / (\text{size} - 1)$ [S1, unofficial].
    *   If `False`: The corners of the grid correspond to the edges of the pixels. The range is $[0, \text{size}]$. The normalization scale is $2 / \text{size}$ [S1, unofficial].
*   **`padding_mode` (str):** Defines behavior for coordinates outside $[-1, 1]$. Common modes are `'zeros'`, `'border'`, and `'reflection'` [S1, unofficial].

### Normalization Formula
Coordinates must be normalized to $[-1, 1]$ before sampling. Let $p$ be a coordinate value (e.g., $x$) and $S$ be the corresponding grid size (e.g., $W$).

If `align_corners` is True:
$$ p_{norm} = \frac{2 \cdot p}{S - 1} - 1 $$

If `align_corners` is False:
$$ p_{norm} = \frac{2 \cdot p}{S} - 1 $$

For 3D inputs with coordinate order $(t, x, y)$, the coordinates are reordered to $(x, y, t)$ to match the grid's spatial layout $(t, x, y)$ or $(H, W, D)$ conventions depending on the implementation [S1, unofficial].

### Steps
1.  **Preprocessing:**
    *   Clone and detach coordinates to ensure gradient flow [S1, unofficial].
    *   Move coordinates to the same device and dtype as the input grid [S1, unofficial].
    *   If the grid has a batch dimension and coordinates do not, unsqueeze the grid or coordinates to match dimensions [S1, unofficial].
    *   If the input is 3D, reorder coordinates from $(t, x, y)$ to $(x, y, t)$ if necessary [S1, unofficial].
    *   Transpose the input grid spatial dimensions if required by the underlying `grid_sample` function (e.g., from $H, W, D$ to $D, H, W$) [S1, unofficial].

2.  **Normalization:**
    *   Extract spatial coordinates from $U$.
    *   Apply the normalization formula based on `align_corners` for each spatial dimension [S1, unofficial].
    *   Concatenate normalized coordinates along the last dimension to form the sampling grid $G$ [S1, unofficial].

3.  **Interpolation:**
    *   Call the underlying bilinear/trilinear sampler (e.g., `F.grid_sample`) with the input grid $I$, sampling grid $G$, `align_corners`, and `padding_mode` [S1, unofficial].
    *   The sampler computes weights for the 4 (2D) or 8 (3D) surrounding grid points and returns the weighted sum [S1, unofficial].

4.  **Post-processing:**
    *   Squeeze any singleton dimensions added during batching [S1, unofficial].
    *   If a validity mask is requested, return a boolean mask where $-1 < x_{norm} < 1$ and $-1 < y_{norm} < 1$ (and $z$ for 3D) [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bilinear_sampler(input_grid, coords, align_corners=True, padding_mode='border'):
    """
    input_grid: Tensor of shape (B, C, H, W) or (B, C, T, H, W)
    coords: Tensor of shape (B, N, 2) or (B, N, 3)
    """
    # 1. Preprocessing
    # Ensure coords are on the same device and dtype
    coords = coords.clone().detach().to(input_grid.device, input_grid.dtype)
    
    # Handle batch dimensions
    if input_grid.dim() == 4 and coords.dim() == 3:
        # Input is (B, C, H, W), Coords is (B, N, 2)
        pass
    elif input_grid.dim() == 5 and coords.dim() == 3:
        # Input is (B, C, T, H, W), Coords is (B, N, 3)
        # Reorder coords from (t, x, y) to (x, y, t) if necessary
        # Assuming coords are (x, y, t) for grid_sample compatibility
        coords = coords[..., [1, 2, 0]] # Reorder if input was (t, x, y)
        
    # Transpose spatial dims for 3D if required by framework (e.g., D, H, W)
    if input_grid.dim() == 5:
        input_grid = input_grid.permute(0, 1, 4, 2, 3) # (B, C, D, H, W) -> (B, C, D, H, W) if already D,H,W? 
        # Note: PyTorch grid_sample expects (B, C, D, H, W) for 3D
        # If input is (B, C, T, H, W), we assume T is D.
        
    # 2. Normalization
    sizes = input_grid.shape[2:] # (H, W) or (D, H, W)
    normalized_coords = []
    
    for i, size in enumerate(sizes):
        coord = coords[..., i]
        if align_corners:
            scale = 2.0 / max(size - 1, 1)
        else:
            scale = 2.0 / size
        norm_coord = coord * scale - 1.0
        normalized_coords.append(norm_coord)
        
    grid = torch.stack(normalized_coords, dim=-1) # (B, N, 2) or (B, N, 3)
    
    # 3. Interpolation
    # Expand grid to match batch if necessary
    # F.grid_sample expects grid of shape (B, H_out, W_out, 2) or (B, D_out, H_out, W_out, 3)
    # If coords are (B, N, 2), we need to reshape to (B, 1, N, 2) for 2D? 
    # Actually, grid_sample expects spatial dimensions. 
    # If we want to sample N points, we treat N as the spatial output size.
    
    # Reshape grid to (B, 1, N, 2) for 2D or (B, 1, 1, N, 3) for 3D?
    # PyTorch grid_sample: grid shape (B, H_out, W_out, 2)
    # If we have N points, we can set H_out=1, W_out=N for 2D? 
    # Or more commonly, coords are (B, H_out, W_out, 2).
    
    # Let's assume coords are (B, N, 2) and we want output (B, C, N)
    # We reshape grid to (B, 1, N, 2)
    if grid.dim() == 3: # (B, N, 2)
        grid = grid.unsqueeze(1) # (B, 1, N, 2)
        
    # Transpose input grid to (B, C, H, W) -> (B, C, W, H) if needed? 
    # PyTorch grid_sample expects (B, C, H, W)
    
    output = torch.nn.functional.grid_sample(
        input_grid, 
        grid, 
        mode='bilinear', 
        padding_mode=padding_mode, 
        align_corners=align_corners
    )
    
    # 4. Post-processing
    # Output shape is (B, C, 1, N) for 2D
    output = output.squeeze(2) # (B, C, N)
    
    return output
```

### Edge Cases and Pitfalls
*   **Coordinate Convention Mismatch:** Using `align_corners=True` with data normalized to $[0, 1]$ without scaling will result in incorrect sampling positions [S1, unofficial].
*   **Dimension Ordering:** 3D grids often require transposing spatial dimensions to match the framework's expected $(D, H, W)$ order [S1, unofficial].
*   **Batch Dimension Handling:** If the batch dimension is missing in coordinates or grid, it must be added using `unsqueeze` to match [S1, unofficial].
*   **Boundary Conditions:** Coordinates outside $[-1, 1]$ are handled by `padding_mode`. `'zeros'` fills with 0, `'border'` replicates edge values, and `'reflection'` reflects the grid [S1, unofficial].

## Subtopics

### [Image Sampling via Grid Sample](15200-image-sampling-via-grid-sample.md)
This method samples image colors at specific 2D pixel coordinates using PyTorch's `grid_sample` function with bilinear interpolation. It serves as the bridge between ray-casting logic (which produces 2D coordinates in normalized space) and the final pixel color values. It differs from the parent [Bilinear Sampler](12826-bilinear-sampler.md) by enforcing a specific coordinate transformation pipeline (negation and permutation) required by the assignment's ray bundle structure, rather than exposing the full general-purpose API.

### Sample Features 4D
Samples spatial features from a 4D tensor at given coordinates using bilinear interpolation, returning features per point [S1, unofficial]. It wraps the standard bilinear sampler and permutes the output from $(B, C, R, 1)$ to $(B, R, C)$ for easier downstream processing [S1, unofficial].

### PointRend Point Sampling
Samples features at specific 2D coordinates using `point_sample`, which wraps `grid_sample` [S6, unofficial]. It normalizes point coordinates from $[0, 1]$ to $[-1, 1]$ and supports uncertainty-based sampling with oversampling and importance sampling ratios [S6, unofficial]. It is used in segmentation tasks to refine predictions at uncertain regions [S6, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of sampling points. The interpolation itself is constant time per point (4 or 8 lookups and weights) [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the output and intermediate grid tensors [S1, unofficial].
*   **Performance:** Highly optimized in deep learning frameworks (e.g., PyTorch, TensorFlow) using CUDA kernels for GPU acceleration [S1, unofficial].
*   **Strengths:** Differentiable, allowing gradients to flow back to the coordinates and the grid values; efficient for batched operations [S1, unofficial].
*   **Limitations:** Requires coordinates to be normalized to $[-1, 1]$; boundary handling can introduce artifacts if not configured correctly [S1, unofficial].
*   **When to Use:** When sampling from dense grids (images, volumes) at arbitrary coordinates in a differentiable manner, such as in spatial transformer networks, ray casting, or feature alignment [S1, unofficial].
*   **When Not to Use:** When sampling from sparse data structures (e.g., point clouds, voxels with large empty spaces) where nearest-neighbor or tree-based sampling is more efficient [S1, unofficial].

## Background
The Bilinear Sampler is a fundamental component in differentiable rendering and spatial transformation networks. It is implemented in major deep learning frameworks such as PyTorch (`torch.nn.functional.grid_sample`) and TensorFlow (`tf.image.gridsample`) [S1, unofficial]. It originated from the need to make spatial transformations and feature sampling differentiable, enabling end-to-end training of models that involve geometric operations [S1, unofficial]. It is widely used in 3D vision, neural rendering, and image registration [S1, unofficial].

## Sources
[S1] InternRobotics/AnySplat, https://github.com/InternRobotics/AnySplat, fetched 2026-10-27 (unofficial)
[S2] YaourtB/GPS_plus, https://github.com/YaourtB/GPS_plus, fetched 2026-10-27 (unofficial)
[S3] tomluetjen/torchskradon, https://github.com/tomluetjen/torchskradon, fetched 2026-10-27 (unofficial)
[S4] xmed-lab/DIF-Gaussian, https://github.com/xmed-lab/DIF-Gaussian, fetched 2026-10-27 (unofficial)
[S5] ElenaViewSynthesis/4dgs, https://github.com/ElenaViewSynthesis/4dgs, fetched 2026-10-28 (unofficial)
[S6] sxyu/pixel-nerf, https://github.com/sxyu/pixel-nerf, fetched 2026-10-26 (unofficial)
