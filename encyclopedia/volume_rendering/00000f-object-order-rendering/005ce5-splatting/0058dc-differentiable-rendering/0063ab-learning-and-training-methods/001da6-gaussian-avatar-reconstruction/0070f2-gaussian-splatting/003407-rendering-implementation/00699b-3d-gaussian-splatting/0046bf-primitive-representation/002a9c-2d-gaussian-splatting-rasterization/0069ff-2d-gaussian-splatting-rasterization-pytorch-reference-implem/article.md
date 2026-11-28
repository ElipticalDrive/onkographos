# 2D Gaussian Splatting Rasterization (PyTorch Reference Implementation)


## Overview
This method provides a PyTorch-based reference implementation for rasterizing 2D Gaussian splats (surfels) into color, alpha, and normal maps. It solves the differentiable rendering problem by leveraging PyTorch's automatic differentiation engine to compute gradients for training, unlike optimized CUDA kernels that require custom C++/CUDA extensions. It sets itself apart from its sibling [Vanilla 2D Gaussian Splatting Rendering](12208-vanilla-2d-gaussian-splatting-rendering.md) by explicitly supporting batched processing and automatic gradient computation, making it suitable for end-to-end learning pipelines, albeit with higher memory consumption and lower speed.

## How it works
The algorithm iteratively accumulates contributions from 2D Gaussians to output tensors. It relies on helper functions `rasterize_to_indices_in_range_2dgs` and `accumulate_2dgs` to manage the rasterization logic and accumulation, respectively. The implementation assumes the presence of the `nerfacc` package for specific accumulation utilities, although the core logic described here is self-contained in PyTorch.

### Inputs and Outputs
**Inputs:**
- `means2d`: Tensor of shape `[C, N, 2]`. 2D mean positions of $N$ Gaussians for $C$ cameras/images.
- `ray_transforms`: Tensor of shape `[C, N, 3, 3]`. Transformation matrices for the Gaussians.
- `colors`: Tensor of shape `[C, N, channels]`. Color values for each Gaussian.
- `normals`: Tensor of shape `[C, N, 3]`. Normal vectors for each Gaussian.
- `opacities`: Tensor of shape `[C, N]`. Opacity values for each Gaussian.
- `isect_offsets`: Tensor of shape `[C, tile_height, tile_width]`. Offsets indicating the start of intersection data for each tile.
- `flatten_ids`: Tensor of shape `[n_isects]`. Flattened list of Gaussian IDs involved in intersections.
- `tile_size`: Integer. Size of the tiles used for rasterization.
- `batch_per_iter`: Integer. Number of batches processed per iteration loop. Default: 100 [S1, unofficial].
- `backgrounds` (Optional): Tensor of background colors.

**Outputs:**
- `render_colors`: Tensor of shape `[C, image_height, image_width, channels]`.
- `render_alphas`: Tensor of shape `[C, image_height, image_width, 1]`.
- `render_normals`: Tensor of shape `[C, image_height, image_width, 3]`.

### Data Structures
The implementation uses standard PyTorch tensors. The `isect_offsets` tensor is used to calculate the range of intersections to process. The `flatten_ids` tensor contains the indices of the Gaussians that intersect with the current view frustum or tiles.

### Steps
1.  **Initialization**: Initialize `render_colors`, `render_alphas`, and `render_normals` to zeros with shapes `[C, image_height, image_width, channels]`, `[C, image_height, image_width, 1]`, and `[C, image_height, image_width, 3]` respectively. These tensors must reside on the same device as `means2d` [S1, unofficial].
2.  **Block Size Calculation**: Calculate `block_size` as `tile_size * tile_size` [S1, unofficial].
3.  **Offset Flattening and Range Calculation**:
    - Flatten `isect_offsets` and append `n_isects` (the total number of intersections) to create `isect_offsets_fl`.
    - Calculate `max_range` as the maximum difference between consecutive elements in `isect_offsets_fl`. This determines the maximum number of intersections to process in a batch [S1, unofficial].
4.  **Batch Count Calculation**: Calculate `num_batches` as $\lceil \text{max\_range} / \text{block\_size} \rceil$ [S1, unofficial].
5.  **Iteration Loop**: Loop over steps from $0$ to `num_batches` with a stride of `batch_per_iter`.
    - **Transmittance Calculation**: Calculate `transmittances` as $1.0 - \text{render\_alphas}$ [S1, unofficial].
    - **Index Retrieval**: Call `rasterize_to_indices_in_range_2dgs` with the current step range, `transmittances`, and input tensors. This returns `gs_ids`, `pixel_ids`, and `camera_ids`, all of shape `[M]`, where $M$ is the number of intersections in the current batch [S1, unofficial].
    - **Early Exit**: If $M = 0$, break the loop [S1, unofficial].
    - **Accumulation**: Call `accumulate_2dgs` with the gathered indices (`gs_ids`, `pixel_ids`, `camera_ids`) and input tensors. This returns `renders_step` (color contribution), `accs_step` (alpha contribution), and `renders_normal_step` (normal contribution) [S1, unofficial].
    - **Update Tensors**:
        - Update `render_colors` by adding `renders_step` multiplied by `transmittances` (broadcasted to channels).
        - Update `render_alphas` by adding `accs_step` multiplied by `transmittances` (broadcasted to 1).
        - Update `render_normals` by adding `renders_normal_step` multiplied by `transmittances` (broadcasted to 3) [S1, unofficial].
6.  **Background Compositing**: If `backgrounds` tensor is provided, add `backgrounds[:, None, None, :]` multiplied by $(1.0 - \text{render\_alphas})$ to `render_colors` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def rasterize_to_pixels_2dgs(
    means2d: torch.Tensor,
    ray_transforms: torch.Tensor,
    colors: torch.Tensor,
    normals: torch.Tensor,
    opacities: torch.Tensor,
    isect_offsets: torch.Tensor,
    flatten_ids: torch.Tensor,
    tile_size: int,
    batch_per_iter: int = 100,
    backgrounds: Optional[torch.Tensor] = None
) -> Tuple[torch.Tensor, torch.Tensor, torch.Tensor]:
    """
    PyTorch reference implementation for 2D Gaussian Splatting rasterization.
    
    Args:
        means2d: [C, N, 2]
        ray_transforms: [C, N, 3, 3]
        colors: [C, N, channels]
        normals: [C, N, 3]
        opacities: [C, N]
        isect_offsets: [C, tile_height, tile_width]
        flatten_ids: [n_isects]
        tile_size: int
        batch_per_iter: int
        backgrounds: Optional [C, channels]
        
    Returns:
        render_colors: [C, H, W, channels]
        render_alphas: [C, H, W, 1]
        render_normals: [C, H, W, 3]
    """
    C, N, _ = means2d.shape
    # Assuming image dimensions are derived from tile_size and tile_grid dimensions
    # Note: The source notes do not explicitly provide H, W, or tile_grid dimensions.
    # We assume they are available or derived from isect_offsets shape.
    # isect_offsets shape is [C, tile_height, tile_width]
    C_iso, tile_h, tile_w = isect_offsets.shape
    H = tile_h * tile_size
    W = tile_w * tile_size
    
    # Initialize output tensors
    render_colors = torch.zeros((C, H, W, colors.shape[-1]), device=means2d.device)
    render_alphas = torch.zeros((C, H, W, 1), device=means2d.device)
    render_normals = torch.zeros((C, H, W, 3), device=means2d.device)
    
    # Calculate block_size
    block_size = tile_size * tile_size
    
    # Flatten isect_offsets and append n_isects
    # isect_offsets_fl shape: [C * tile_h * tile_w + 1]
    isect_offsets_fl = isect_offsets.flatten()
    n_isects = flatten_ids.shape[0]
    isect_offsets_fl = torch.cat([isect_offsets_fl, torch.tensor([n_isects], device=means2d.device)])
    
    # Calculate max_range
    # Differences between consecutive elements
    diffs = isect_offsets_fl[1:] - isect_offsets_fl[:-1]
    max_range = torch.max(diffs).item()
    
    # Calculate num_batches
    num_batches = math.ceil(max_range / block_size)
    
    # Loop over batches
    for step in range(0, num_batches, batch_per_iter):
        # Calculate transmittances
        transmittances = 1.0 - render_alphas
        
        # Determine range for this batch
        start_idx = step * block_size
        end_idx = min((step + batch_per_iter) * block_size, max_range)
        
        # Call rasterize_to_indices_in_range_2dgs
        # This function is assumed to be defined elsewhere or provided by the library
        # It returns gs_ids, pixel_ids, camera_ids of shape [M]
        gs_ids, pixel_ids, camera_ids = rasterize_to_indices_in_range_2dgs(
            means2d, ray_transforms, colors, normals, opacities,
            isect_offsets, flatten_ids, start_idx, end_idx
        )
        
        M = gs_ids.shape[0]
        if M == 0:
            break
            
        # Call accumulate_2dgs
        # This function is assumed to be defined elsewhere or provided by the library
        # It returns renders_step, accs_step, renders_normal_step
        renders_step, accs_step, renders_normal_step = accumulate_2dgs(
            gs_ids, pixel_ids, camera_ids,
            means2d, ray_transforms, colors, normals, opacities,
            transmittances
        )
        
        # Update render_colors
        # renders_step shape: [M, channels]
        # transmittances shape: [C, H, W, 1]
        # We need to scatter add based on pixel_ids and camera_ids
        # Note: The source notes say "adding renders_step multiplied by transmittances"
        # This implies a scatter operation or gathering.
        # For a pure PyTorch implementation without custom CUDA, this would typically involve
        # scatter_add_ or similar. The source notes simplify this to "Update ... by adding".
        # We assume accumulate_2dgs returns values ready to be added to the specific pixels.
        
        # Re-reading source: "Call accumulate_2dgs with the gathered indices and tensors to get renders_step..."
        # "Update render_colors by adding renders_step multiplied by transmittances"
        # This suggests accumulate_2dgs might return values aligned with the output tensor or
        # the update logic handles the indexing. Given the constraint of "no placeholders",
        # and the fact that accumulate_2dgs is a helper, we describe the update as per source.
        
        # To make this implementable, we assume accumulate_2dgs returns tensors that can be
        # directly added or that the update logic is a scatter_add.
        # However, the source says "Update render_colors by adding...".
        # Let's assume the helper functions handle the indexing internally or return
        # tensors of shape [C, H, W, ...] for the step.
        # But the source says "gs_ids, pixel_ids, camera_ids (all shape [M])".
        # And "renders_step" is returned.
        # It is most likely that accumulate_2dgs performs the scatter_add internally or
        # returns the contributions to be scattered.
        # Given the ambiguity, we will write the update as a scatter_add operation
        # if accumulate_2dgs returns raw contributions, or direct addition if it returns
        # partial framebuffers.
        # The source says "Update render_colors by adding renders_step multiplied by transmittances".
        # This phrasing usually implies direct addition if shapes match, or scatter if they don't.
        # Since renders_step comes from indices of shape [M], it likely has shape [M, ...].
        # Therefore, we must scatter.
        
        # However, to avoid guessing the internal implementation of accumulate_2dgs,
        # we will stick to the high-level description provided in the source notes,
        # noting that the actual PyTorch code would use scatter_add.
        
        # Let's assume accumulate_2dgs returns tensors of shape [M, channels], [M, 1], [M, 3]
        # Then we scatter them into render_colors, render_alphas, render_normals.
        
        # Transmittances need to be gathered for the specific pixels
        # pixel_ids shape [M], camera_ids shape [M]
        # transmittances shape [C, H, W, 1]
        # We need transmittances[pixel_ids, camera_ids]
        
        # Gather transmittances
        # Note: pixel_ids are likely linear indices or (y, x).
        # Assuming pixel_ids are linear indices into the flattened [C, H, W] space?
        # Or (camera, y, x)?
        # The source says "pixel_ids ... shape [M]".
        # Let's assume they are linear indices into the output tensor's spatial dimensions
        # combined with camera index.
        
        # This level of detail for the helper functions is not provided in the source notes.
        # We will mark the helper calls as defined in the library.
        
        # Update logic as per source:
        # render_colors += renders_step * transmittances
        # This implies renders_step is already weighted or the multiplication is element-wise
        # after broadcasting.
        
        # Given the constraint "Pseudocode has no placeholders", we must write out the scatter.
        # But we don't know the exact format of pixel_ids.
        # We will assume pixel_ids are linear indices into the [C, H, W] space.
        
        # Flatten output tensors for scattering
        render_colors_flat = render_colors.view(C * H * W, -1)
        render_alphas_flat = render_alphas.view(C * H * W, -1)
        render_normals_flat = render_normals.view(C * H * W, -1)
        
        # Gather transmittances for the specific pixels
        # pixel_ids are assumed to be linear indices into C*H*W
        transmittances_gathered = transmittances.view(C * H * W, 1)[pixel_ids]
        
        # Weighted contributions
        weighted_colors = renders_step * transmittances_gathered
        weighted_alphas = accs_step * transmittances_gathered
        weighted_normals = renders_normal_step * transmittances_gathered
        
        # Scatter add
        render_colors_flat.scatter_add_(0, pixel_ids.unsqueeze(1).expand(-1, weighted_colors.shape[1]), weighted_colors)
        render_alphas_flat.scatter_add_(0, pixel_ids.unsqueeze(1).expand(-1, weighted_alphas.shape[1]), weighted_alphas)
        render_normals_flat.scatter_add_(0, pixel_ids.unsqueeze(1).expand(-1, weighted_normals.shape[1]), weighted_normals)
        
        # Reshape back
        render_colors = render_colors_flat.view(C, H, W, -1)
        render_alphas = render_alphas_flat.view(C, H, W, -1)
        render_normals = render_normals_flat.view(C, H, W, -1)

    # Background compositing
    if backgrounds is not None:
        # backgrounds shape [C, channels]
        # render_alphas shape [C, H, W, 1]
        # Add backgrounds * (1 - render_alphas)
        render_colors = render_colors + backgrounds[:, None, None, :] * (1.0 - render_alphas)
        
    return render_colors, render_alphas, render_normals
```

### Parameters
- `tile_size`: The size of the tiles used for rasterization, used to calculate `block_size` [S1, unofficial].
- `batch_per_iter`: The number of batches processed in each iteration loop. Default value is 100 [S1, unofficial].

### Edge Cases and Pitfalls
- **Performance**: The implementation is significantly slower and consumes more GPU memory than the fully fused CUDA version because it relies on PyTorch's autograd for backpropagation [S1, unofficial].
- **Dependencies**: Requires the `nerfacc` package to be installed, specifically the functions `accumulate_along_rays` and `render_weight_from_alpha` [S1, unofficial]. Note: The pseudocode above uses `rasterize_to_indices_in_range_2dgs` and `accumulate_2dgs` as described in the source notes. If `nerfacc` functions are required, they should be integrated into the accumulation step.
- **Precision**: Standard floating-point precision issues may arise during accumulation, especially with large numbers of splats.
- **Boundaries**: Ensure that `isect_offsets` and `flatten_ids` are correctly aligned with the tile structure.

## Subtopics

### [Per-Primitive Texture Mapping for Reflective Gaussian Splatting](8629-per-primitive-texture-mapping-for-reflective-gaussian-splatt.md)
Adds texture mapping to reflective Gaussians. Choose when rendering reflective surfaces with high-frequency details.

### [ToF Gaussian Rasterization Forward Pass](9261-tof-gaussian-rasterization-forward-pass.md)
Rasterizes Gaussians for Time-of-Flight cameras. Choose when simulating or training with ToF sensor data.

### [Vanilla 2D Gaussian Splatting Rendering](12208-vanilla-2d-gaussian-splatting-rendering.md)
Basic NumPy reference rasterization. Choose for debugging or when PyTorch overhead is unacceptable.

### [Reflective Gaussian Splatting](12321-reflective-gaussian-splatting.md)
General reflective Gaussian splatting. Choose when rendering general reflective materials.

### [Nexels](13877-nexels.md)
A specific method for neural texturing. Choose when integrating neural textures with Gaussian splatting.

## Operating characteristics
- **Time Complexity**: $O(N \cdot M)$ where $N$ is the number of Gaussians and $M$ is the number of intersections, with additional overhead from Python/PyTorch interpretation.
- **Memory Complexity**: $O(C \cdot H \cdot W \cdot \text{channels})$ for the output tensors, plus intermediate tensors for indices and transmittances.
- **Measured Performance**: Significantly slower than CUDA implementations. Memory usage is higher due to autograd graph storage [S1, unofficial].
- **Scaling**: Scales with batch size and image resolution, but limited by GPU memory due to the creation of large intermediate tensors.
- **Strengths**: Easy to implement, debug, and modify. Supports automatic differentiation out-of-the-box.
- **Limitations**: Slow execution speed. High memory consumption. Requires `nerfacc` package.
- **When to Use**: For research, prototyping, and training where ease of gradient computation is more important than inference speed.
- **When Not to Use**: For real-time inference or production deployment where performance is critical.

## Background
This implementation is part of the GS-Scale project, an unofficial PyTorch reference for 2D Gaussian Splatting rasterization. It is designed to facilitate learning and training methods for Gaussian Avatar Reconstruction by providing a differentiable rendering pipeline.

## Sources
[S1] SNU-ARC/GS-Scale, https://github.com/SNU-ARC/GS-Scale, fetched 2026-10-27 (unofficial)
