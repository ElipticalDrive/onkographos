# 3D Gaussian Splatting Rasterization (GPU)

Also called: 3DGS Rasterization, GSplat rasterizer, Gaussian Rasterization (GPU), Gaussian Splatting Rasterizer, GaussianRasterizer, diff-gaussian-rasterization

## Overview
A differentiable rasterization method for 3D Gaussians that sorts Gaussians by tile and depth to perform alpha blending in parallel per tile [S2, S10, unofficial]. It solves the problem of efficiently rendering millions of 3D Gaussian splats to a 2D screen while maintaining exact gradients for all Gaussian parameters (means, opacities, scales, rotations, spherical harmonics) via a CUDA backend [S10, unofficial]. It sets itself apart from its parent [Tiled 2D Gaussian Splatting Rasterization](2061-tiled-2d-gaussian-splatting-rasterization.md) by implementing the specific sorting, binning, and tile-based parallel rendering kernels required for the standard 3DGS forward and backward passes, often exposed as a `torch.autograd.Function` [S4, S10, unofficial].

## How it works
**Inputs and Outputs:**
The method takes $P$ 3D Gaussians defined by 3D means $\mathbf{\mu}_{3D}$, scales $\mathbf{s}$, rotations $\mathbf{q}$ (quaternions), opacities $\alpha$, and color representations (Spherical Harmonics coefficients $\mathbf{SH}$ or direct colors $\mathbf{c}$) [S10, unofficial]. Camera parameters include view matrix $\mathbf{V}$, projection matrix $\mathbf{P}$, and field of view tangents $\tan \theta_x, \tan \theta_y$ [S10, unofficial].
Outputs are the rendered RGB image $\mathbf{C} \in \mathbb{R}^{H \times W \times 3}$, radii $\mathbf{r} \in \mathbb{Z}^P$, and optionally depth/weight buffers [S10, unofficial]. The backward pass returns gradients for all input parameters [S10, unofficial].

**Data Structures:**
The implementation relies on three main GPU buffers managed by the C++/CUDA backend:
1. **GeometryState**: Stores per-Gaussian precomputed data [S11, unofficial].
   - `depths`: `float[P]`
   - `means2D`: `float2[P]` (screen-space center)
   - `cov3D`: `float[P*6]` (upper triangle of 3x3 3D covariance)
   - `rgb`: `float[P*3]` (precomputed color)
   - `conic_opacity`: `float4[P]` (x,y,z=conic matrix components, w=opacity)
   - `internal_radii`: `int[P]`
   - `tiles_touched`: `uint32_t[P]`
   - `point_offsets`: `int[P]` (prefix sum of tiles touched)
2. **BinningState**: Stores sorting keys and values [S11, unofficial].
   - `point_list_keys_unsorted`: `uint64_t[P]` (Key: high 32 bits = tile ID, low 32 bits = depth)
   - `point_list_unsorted`: `uint32_t[P]` (Value: Gaussian index)
   - `point_list_keys`: `uint64_t[P]` (Sorted keys)
   - `point_list`: `uint32_t[P]` (Sorted indices)
3. **ImageState**: Stores per-pixel state.
   - `accum_alpha`: `float[H*W]`
   - `n_contrib`: `int[H*W]`
   - `ranges`: `uint2[H*W]` (start/end indices in sorted `point_list` for each tile/pixel)

**Parameters:**
- `BLOCK_X`, `BLOCK_Y`: Tile dimensions in pixels. Typical value: 16x16 [S2, unofficial].
- `BLOCK_SIZE`: Threads per block. Typical value: 256 [S2, unofficial].
- `NUM_CHANNELS`: 3 (RGB) [S2, unofficial].
- `scale_modifier`: Multiplier for scale components. Typical value: 1.0 [S2, unofficial].

**Steps:**

1. **Preprocessing (Projection & Conic Computation):**
   For each Gaussian $i$:
   a. **Frustum Culling**: Keep only Gaussians with a 99% confidence interval intersecting the view frustum [S3].
   b. **3D Covariance**: Compute 3D covariance matrix $\Sigma_{3D}$ from scale $\mathbf{s}$ and rotation $\mathbf{q}$ [S2, unofficial]. If precomputed, use it. Apply `scale_modifier` [S2, unofficial].
   c. **Jacobian**: Compute the Jacobian $\mathbf{J}$ of the affine approximation of the projective transformation [S2, unofficial]. Let $\mathbf{W}$ be the rotation part of the view matrix [S2, unofficial]. The transformation matrix is $\mathbf{T} = \mathbf{W}\mathbf{J}$ [S2, unofficial].
   d. **2D Covariance**: Compute screen-space covariance $\Sigma_{2D} = \mathbf{T} \Sigma_{3D} \mathbf{T}^T$ [S2, unofficial].
   e. **Low-Pass Filter**: Add 0.3 to the diagonal elements of $\Sigma_{2D}$ to ensure minimum pixel width/height [S2, unofficial].
   f. **Conic Matrix**: Invert $\Sigma_{2D}$ to get the conic matrix $\mathbf{C}_{2D} = \Sigma_{2D}^{-1}$ [G]. Store as `conic_opacity` where $x=C_{00}, y=C_{01}, z=C_{11}, w=\alpha$ [S2, unofficial].
   g. **Radius**: Compute eigenvalues $\lambda_{1,2}$ of $\Sigma_{2D}$:
      $$ \lambda_{1,2} = 0.5 \left( (\Sigma_{2D})_{00} + (\Sigma_{2D})_{11} \right) \pm \sqrt{\max(0.1f, 0.25 \left( (\Sigma_{2D})_{00} - (\Sigma_{2D})_{11} \right)^2 + (\Sigma_{2D})_{01}^2)} $$
      Calculate radius $r = \lceil 3 \sqrt{\max(\lambda_1, \lambda_2)} \rceil$ [S2, unofficial].
   h. **Tile Bounding Box**: Determine the tiles touched by the Gaussian's bounding box [S2, unofficial].
   i. **Key Generation**: For each touched tile, generate a 64-bit key: `(tile_id << 32) | depth_packed`. Store Gaussian index as value. If $r \le 0$, skip [S11, unofficial].

2. **Sorting and Binning:**
   a. **Prefix Sum**: Compute an inclusive prefix sum over `tiles_touched` to determine offsets for writing to the binning buffer [S2, S11, unofficial].
   b. **Radix Sort**: Sort the generated key/value pairs using CUDA Radix Sort (`cub::DeviceRadixSort::SortPairs`) to order Gaussians first by tile ID and then by depth [S2, unofficial].
   c. **Range Calculation**: Identify start and end indices in the sorted list for each tile by comparing tile IDs of consecutive keys. Store these ranges in `ImageState.ranges` [S2, unofficial].

3. **Rasterization (Tile-Parallel Rendering):**
   Each thread block handles one tile.
   a. **Load Range**: Load the range of points for the tile from the sorted `point_list` [S2, unofficial].
   b. **Batch Processing**: Iterate over Gaussians in batches. Collectively fetch Gaussian data (ID, xy, conic_opacity) from global to shared memory [S2, unofficial].
   c. **Pixel Loop**: For each pixel $(u,v)$ in the tile:
      i. Initialize transmittance and color buffers [G].
      ii. For each Gaussian $j$ in the tile's range:
          1. Compute displacement $\mathbf{d} = \mathbf{xy}_j - \mathbf{pix}_{uv}$ [S2, unofficial].
          2. Compute power:
             $$ \text{power} = -0.5 \left( C_{00} d_x^2 + C_{11} d_y^2 \right) - C_{01} d_x d_y $$
          3. If $\text{power} > 0$, continue (outside ellipse) [S2, unofficial].
          4. Compute alpha: $\alpha_j = \min(0.99, \alpha_j^{\text{orig}} \cdot e^{\text{power}})$ [S2, unofficial].
          5. If $\alpha_j < 1/255$, continue [G].
          6. Composite:
             $$ C \leftarrow C + \alpha_j \cdot \mathbf{color}_j \cdot T $$
             $$ T \leftarrow T \cdot (1 - \alpha_j) $$
          7. If $T < 0.0001$, mark done for this pixel [G].
   d. **Write Output**: Write final $T$, contribution count, and $C$ to output buffers [S2, unofficial].

4. **Backward Pass:**
   The backward pass restores saved tensors from the forward pass (`ctx.save_for_backward`) and calls the C++ backward kernel `_C.rasterize_gaussians_backward` [S4, S10, unofficial]. It computes gradients for `means3D`, `means2D`, `dc`, `sh`, `opacities`, `scales`, `rotations`, and `cov3Ds_precomp` [S10, unofficial]. The exact gradient formulas are implemented in the CUDA kernel, utilizing the saved `radii`, `geomBuffer`, `binningBuffer`, and `imgBuffer` [S10, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def rasterize_gaussians_forward(means3D, scales, rotations, opacities, shs, dc, 
                                viewmatrix, projmatrix, tanfovx, tanfovy, 
                                bg, scale_modifier, width, height, sh_degree):
    """
    Forward pass for 3D Gaussian Splatting rasterization.
    """
    P = means3D.shape[0]
    H, W = height, width
    
    # Step 1: Preprocessing (Projection & Conic Computation)
    # Compute 3D covariance from scales/rotations or use cov3Ds_precomp
    # Transform to 2D using Jacobian of projection
    # Compute conic matrix and radius
    # Store in GeometryState: means2D, depths, cov3D, rgb, conic_opacity, internal_radii, tiles_touched
    # (CUDA kernel: preprocessCUDA)
    
    # Step 2: Generate Keys and Values
    # For each Gaussian, determine tiles touched based on radius
    # Create keys: (tile_id << 32) | depth_packed
    # Create values: gaussian_index
    # Store in BinningState: point_list_keys_unsorted, point_list_unsorted
    # (CUDA kernel: duplicateWithKeys)
    
    # Step 3: Sort (cub::DeviceRadixSort::SortPairs)
    # Sort keys and values to order Gaussians by tile ID then depth
    # (CUDA kernel: cub::DeviceRadixSort::SortPairs)
    
    # Step 4: Compute Ranges (computeRangesKernel)
    # Identify start/end indices for each tile in sorted list
    # (CUDA kernel: computeRangesKernel)
    
    # Step 5: Render (renderCUDA kernel)
    # Launch tile-parallel kernel
    tile_grid_x = (W + 15) // 16
    tile_grid_y = (H + 15) // 16
    # renderCUDA<<<(tile_grid_x, tile_grid_y, 1), (16, 16, 1)>>>(...)
    # Returns color, radii, invdepths, accum_weights
    
    return color, radii, invdepths, accum_weights

def rasterize_gaussians_backward(grad_out_color, saved_tensors):
    """
    Backward pass for 3D Gaussian Splatting rasterization.
    """
    # Restore saved tensors
    means3D, scales, rotations, opacities, shs, dc, radii, geomBuffer, binningBuffer, imgBuffer = saved_tensors
    
    # Call backward CUDA kernel
    # _C.rasterize_gaussians_backward(...)
    # Returns gradients for all input parameters
    # The exact gradient computation is implemented in the C++/CUDA extension.
    grad_means3D, grad_means2D, grad_dc, grad_sh, grad_opacities, grad_scales, grad_rotations = _C.rasterize_gaussians_backward(
        grad_out_color, means3D, scales, rotations, opacities, shs, dc, radii, geomBuffer, binningBuffer, imgBuffer
    )
    
    return grad_means3D, grad_means2D, grad_dc, grad_sh, grad_opacities, grad_scales, grad_rotations
```

## Subtopics

### [FastGS Residual Pyramid Rendering](13487-fastgs-residual-pyramid-rendering.md)
Multi-scale technique decomposing the image into a Laplacian pyramid of Gaussian splats for independent optimization and rendering at different resolutions.

### [MiniGS Auxiliary & Depth Rendering Interface](13963-minigs-auxiliary-depth-rendering-interface.md)
Validation and dispatch layer for MiniGS-specific rasterizer entry points, enabling auxiliary statistics for blur-aware splitting and depth back-sampling during training.

### [GaussianRasterizer_w_pose](15269-gaussianrasterizer-w-pose.md)
Variant accepting camera rotation and translation deltas to render with pose perturbations, returning pixel coverage statistics.

### [PyTorch Rasterization Wrapper](18512-pytorch-rasterization-wrapper.md)
A `torch.autograd.Function` that bridges Python and a C++ rasterization kernel for 3D Gaussian Splatting, handling argument restructuring and buffer management.

### [Gaussian Rasterization via OpenGL Interop](19822-gaussian-rasterization-via-opengl-interop.md)
Renders 3D Gaussians by leveraging OpenGL's fixed-function point rendering pipeline via CUDA-OpenGL interop, offloading fragment processing to the GPU driver's OpenGL implementation.

## Operating characteristics
- **Time Complexity**: Not given in the sources. [G]
- **Memory Complexity**: Not given in the sources. [G]
- **Performance**: Highly optimized for GPU parallelism. Tile-based approach ensures memory coalescing and efficient use of shared memory. Typical tile size is 16x16 pixels [S2, unofficial].
- **Strengths**: Differentiable, fast, supports millions of Gaussians, exact gradients.
- **Limitations**: Requires significant VRAM for large scenes due to sorting buffers. Depth packing assumes depth fits in 32 bits [S2, unofficial].
- **When to use**: Standard 3DGS rendering and training. [G]
- **When not to use**: If VRAM is extremely limited, consider [Non-Tiled 2D Gaussian Splatting with Transmittance Culling](2067-non-tiled-2d-gaussian-splatting-with-transmittance-culling.md) or other memory-efficient variants. [G]

## Background
Originated from the 3D Gaussian Splatting paper by Kerbl et al. (2023) [S1, unofficial][S3][S5, S10, unofficial]. The method was developed to enable real-time differentiable rendering of 3D scenes represented as Gaussian splats [S2, unofficial]. It builds on the tile-based rasterization techniques used in traditional graphics pipelines but adapts them for the specific needs of Gaussian splatting, including conic matrix approximation and alpha blending [S2, unofficial]. The implementation is typically provided as a custom PyTorch autograd function wrapping a C++/CUDA backend [S2, unofficial][S3][S4, S10, unofficial].

## Sources
[S1] Polycam released web 3D Gaussian Splatting Viewer, https://poly.cam/gaussian-splatting, fetched 2026-10-26 (unofficial)
[S2] DBauer15/gscache, https://github.com/DBauer15/gscache, fetched 2026-10-26 (unofficial)
[S3] 3D Gaussian Splatting for Real-Time Radiance Field Rendering, https://arxiv.org/abs/2308.04079, fetched 2026-10-26
[S4] dendenxu/fast-gaussian-rasterization, https://github.com/dendenxu/fast-gaussian-rasterization, fetched 2026-10-26 (unofficial)
[S5] yuyouxixi/x2-gaussian, https://github.com/yuyouxixi/x2-gaussian, fetched 2026-10-26 (unofficial)
[S6] SorenZhang574/Laplacian-GS, https://github.com/SorenZhang574/Laplacian-GS, fetched 2026-10-27 (unofficial)
[S7] notlasandu/spirulae-splat, https://github.com/notlasandu/spirulae-splat, fetched 2026-10-27 (unofficial)
[S8] city-super/GSDF, https://github.com/city-super/GSDF, fetched 2026-10-27 (unofficial)
[S9] horizon-research/Fov-3DGS, https://github.com/horizon-research/Fov-3DGS, fetched 2026-10-27 (unofficial)
[S10] XiaoBin2001/Improved-GS, https://github.com/XiaoBin2001/Improved-GS, fetched 2026-10-27 (unofficial)
[S11] otakuxiang/gaussian-splatting-jittor, https://github.com/otakuxiang/gaussian-splatting-jittor, fetched 2026-10-27 (unofficial)
[S12] LiuXiangrui/CompGS, https://github.com/LiuXiangrui/CompGS, fetched 2026-10-27 (unofficial)
[S13] Render depth map, https://github.com/city-super/Scaffold-GS/issues/21, fetched 2026-10-27 (unofficial)
