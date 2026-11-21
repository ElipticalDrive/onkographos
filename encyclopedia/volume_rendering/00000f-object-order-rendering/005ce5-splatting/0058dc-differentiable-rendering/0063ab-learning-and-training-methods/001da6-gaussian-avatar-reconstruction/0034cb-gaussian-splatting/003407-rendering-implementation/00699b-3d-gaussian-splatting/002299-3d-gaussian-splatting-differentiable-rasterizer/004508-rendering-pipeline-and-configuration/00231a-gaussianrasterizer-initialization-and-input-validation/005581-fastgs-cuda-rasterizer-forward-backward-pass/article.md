# FastGS CUDA Rasterizer Forward/Backward Pass

Also called: CUDA Gaussian Rasterization, Fast Gaussian Splatting, FastGS CUDA Rasterizer, FastGS Rasterization Pipeline, FastGS Rasterizer

## Overview
The FastGS CUDA Rasterizer is a CUDA-accelerated differentiable rasterization method for 3D Gaussians [S1, unofficial]. It extends the original differentiable Gaussian rasterization with support for additional buffers (`sampleBuffer`, `accum_metric_counts`) and debug modes [S1, unofficial]. It distinguishes itself from standard implementations by using tile-based binning, radix sorting, and bucket-based parallel blending to optimize memory access and compute throughput [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:** `means3D`, `colors_precomp`, `opacities`, `scales`, `rotations`, `scale_modifier`, `cov3Ds_precomp`, `metric_map`, `viewmatrix`, `projmatrix`, `tanfovx`, `tanfovy`, `image_height`, `image_width`, `dc`, `sh`, `sh_degree`, `campos`, `mult`, `prefiltered`, `debug`, `get_flag` [S1, unofficial].
**Outputs:** `render` (image tensor), `viewspace_points` (gradient tensor), `visibility_filter` (nonzero indices of radii > 0), `radii` (tensor), `accum_metric_counts` (tensor) [S1, unofficial].

### Data Structures
*   **GaussianRasterizationSettings**: NamedTuple fields: `image_height` (int), `image_width` (int), `tanfovx` (float), `tanfovy` (float), `bg` (torch.Tensor), `scale_modifier` (float), `viewmatrix` (torch.Tensor), `projmatrix` (torch.Tensor), `sh_degree` (int), `campos` (torch.Tensor), `mult` (float), `prefiltered` (bool), `debug` (bool), `get_flag` (bool), `metric_map` (torch.Tensor) [S1, unofficial].
*   **SampleState**: Stores bucket-to-tile mappings, tile boundaries (`T`), and aspect ratios (`ar`) [S1, unofficial].
*   **ImgState**: Stores tile ranges, bucket offsets, accumulated alpha, contribution counts (`n_contrib`, `max_contrib`), pixel colors, and scan buffers [S1, unofficial].
*   **BinningState**: Stores the sorted point list (`point_list`) and unsorted keys/values [S1, unofficial].
*   **GeomState**: Stores 2D means (`means2D`), conic matrices (`conic_opacity`), and clamping flags [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def fastgs_forward(means3D, colors_precomp, opacities, scales, rotations, 
                   cov3Ds_precomp, viewmatrix, projmatrix, tanfovx, tanfovy, 
                   image_height, image_width, bg, scale_modifier, dc, sh, 
                   sh_degree, campos, mult, prefiltered, debug, get_flag, 
                   metric_map=None):
    # 1. Initialize screenspace_points tensor with shape (num_gaussians, 4) and requires_grad=True
    screenspace_points = torch.zeros(means3D.shape[0], 4, requires_grad=True, device=means3D.device)
    
    # 2. Calculate focal lengths from image dimensions and tangent field-of-view
    focal_y = image_height / (2.0 * tanfovy)
    focal_x = image_width / (2.0 * tanfovx)
    
    # 3. Initialize metric_map if None
    if metric_map is None:
        metric_map = torch.zeros(image_height * image_width, dtype=torch.int, device=means3D.device)
        
    # 4. Configure GaussianRasterizationSettings
    raster_settings = {
        'image_height': image_height, 'image_width': image_width,
        'tanfovx': tanfovx, 'tanfovy': tanfovy, 'bg': bg,
        'scale_modifier': scale_modifier, 'viewmatrix': viewmatrix,
        'projmatrix': projmatrix, 'sh_degree': sh_degree, 'campos': campos,
        'mult': mult, 'prefiltered': prefiltered, 'debug': debug,
        'get_flag': get_flag, 'metric_map': metric_map
    }
    
    # 5. Handle Covariance and Color Precomputation
    if raster_settings.get('compute_cov3D_python', False):
        cov3D_precomp = compute_cov3D_from_scales_rotations(scales, rotations)
    else:
        cov3D_precomp = cov3Ds_precomp
        
    if raster_settings.get('convert_SHs_python', False):
        colors_precomp = evaluate_SHs(dc, sh, sh_degree, means3D)
    else:
        pass # Pass dc and sh to kernel

    # 6. Prepare Args for CUDA Kernel
    args = (
        raster_settings['bg'], means3D, colors_precomp, opacities, 
        scales, rotations, scale_modifier, cov3D_precomp, metric_map, 
        viewmatrix, projmatrix, tanfovx, tanfovy, image_height, image_width, 
        dc, sh, sh_degree, campos, mult, prefiltered, debug, get_flag
    )
    
    # 7. Invoke CUDA Kernel
    rendered_image, radii, accum_metric_counts = render_fastgs_cuda(*args)
    
    return {
        'render': rendered_image,
        'viewspace_points': screenspace_points,
        'visibility_filter': radii.nonzero().squeeze(),
        'radii': radii,
        'accum_metric_counts': accum_metric_counts
    }

def fastgs_backward(grad_out_color, context_tensors):
    # 1. Restore saved tensors (colors_precomp, means3D, scales, rotations, cov3Ds_precomp, radii, dc, sh, geomBuffer, binningBuffer, imgBuffer, sampleBuffer) and raster_settings from context
    (colors_precomp, means3D, scales, rotations, cov3Ds_precomp, 
     radii, dc, sh, geomBuffer, binningBuffer, imgBuffer, sampleBuffer) = context_tensors[:12]
    raster_settings = context_tensors[12]
    
    # 2. Restructure backward arguments
    args_bw = (
        raster_settings['bg'], means3D, radii, colors_precomp, scales, 
        rotations, raster_settings['scale_modifier'], cov3Ds_precomp, 
        raster_settings['viewmatrix'], raster_settings['projmatrix'], 
        raster_settings['tanfovx'], raster_settings['tanfovy'], grad_out_color, 
        dc, sh, raster_settings['sh_degree'], raster_settings['campos'], 
        geomBuffer, context_tensors['num_rendered'], binningBuffer, 
        context_tensors['num_buckets'], sampleBuffer, raster_settings['debug']
    )
    
    # 3. Invoke backward kernel to compute gradients for means2D, colors_precomp, opacities, means3D, cov3Ds_precomp, dc, sh, scales, rotations
    grads = rasterize_gaussians_backward_cuda(*args_bw)
    return grads

def render_fastgs_cuda(bg, means3D, colors_precomp, opacities, scales, rotations, 
                       scale_modifier, cov3Ds_precomp, metric_map, viewmatrix, 
                       projmatrix, tanfovx, tanfovy, image_height, image_width, 
                       dc, sh, sh_degree, campos, mult, prefiltered, debug, get_flag):
    N = means3D.shape[0]
    
    # 1. Mark visible Gaussians: checkFrustum kernel
    # Transform means to view space using 4x3 matrix (rotation + translation)
    means_view = transform_point_4x3(viewmatrix, means3D)
    # Check frustum: p_view.z > 0.2f
    in_frustum = means_view[:, 2] > 0.2
    present = in_frustum.int()
    
    # 2. Preprocess Gaussians: FORWARD::preprocess kernel
    # Transform to screen space, compute covariances, clamp, determine tiles_touched
    means2D, conic_opacity, tiles_touched = preprocess_gaussians(
        means3D, cov3Ds_precomp, opacities, viewmatrix, projmatrix, 
        tanfovx, tanfovy, image_height, image_width, scale_modifier
    )
    
    # 3. Compute prefix sums: cub::DeviceScan::InclusiveSum on tiles_touched
    point_offsets = inclusive_sum(tiles_touched)
    
    # 4. Duplicate and Key Generation: duplicateWithKeys kernel
    # Generate 64-bit key: (tile_id << 32) | depth
    gaussian_keys_unsorted, gaussian_values_unsorted = duplicate_with_keys(
        present, tiles_touched, point_offsets, means2D
    )
    
    # 5. Radix Sort: cub::DeviceRadixSort::SortPairs
    # Sort range determined by num_rendered. Bit depth = 32 + MSB(tile_count)
    num_rendered = present.sum()
    sorted_keys, sorted_values = radix_sort_pairs(gaussian_keys_unsorted, gaussian_values_unsorted, num_rendered)
    
    # 6. Identify Tile Ranges: identifyTileRanges kernel
    ranges = identify_tile_ranges(sorted_keys)
    
    # 7. Bucket Counting: perTileBucketCount kernel
    # num_buckets = (num_splats + 31) / 32
    bucket_count = (ranges.diff() + 31) // 32
    
    # 8. Bucket Offset Calculation: cub::DeviceScan::InclusiveSum on bucket_count
    bucket_offsets = inclusive_sum(bucket_count)
    
    # 9. Parallel Rendering: FORWARD::render kernel
    # Initialize sample buffer state
    sample_chunk_size = required_size_for_buckets(bucket_offsets[-1] if len(bucket_offsets) > 0 else 0)
    sample_chunkptr = allocate_buffer(sample_chunk_size)
    sample_state = SampleState.from_chunk(sample_chunkptr, bucket_offsets[-1] if len(bucket_offsets) > 0 else 0)
    
    # Determine feature pointer
    if colors_precomp is not None:
        feature_ptr = colors_precomp
    else:
        feature_ptr = geom_state.rgb # From geomBuffer
        
    # Launch render kernel
    out_color = torch.zeros(image_height, image_width, 3, device=means3D.device)
    radii = torch.zeros(N, device=means3D.device)
    metricCount = torch.zeros(image_height * image_width, device=means3D.device)
    
    render_kernel(
        tile_grid=(image_width // BLOCK_X, image_height // BLOCK_Y, 1),
        block=(BLOCK_X, BLOCK_Y, 1),
        ranges=ranges,
        point_list=sorted_values,
        bucket_offsets=bucket_offsets,
        sample_state=sample_state,
        means2D=means2D,
        feature_ptr=feature_ptr,
        conic_opacity=conic_opacity,
        bg=bg,
        out_color=out_color,
        radii=radii,
        metric_map=metric_map,
        get_flag=get_flag,
        metricCount=metricCount
    )
    
    return out_color, radii, metricCount

def preprocess_gaussians(means3D, cov3Ds_precomp, opacities, viewmatrix, projmatrix, 
                         tanfovx, tanfovy, image_height, image_width, scale_modifier):
    # Transform 3D means to view space
    means_view = transform_point_4x3(viewmatrix, means3D)
    # Project to NDC using 4x4 matrix
    means_hom = transform_point_4x4(projmatrix, means_view)
    # Handle near-zero w by adding 0.0000001f
    w = means_hom[:, 3] + 1e-7
    means_ndc = means_hom[:, :3] / w.unsqueeze(1)
    
    # NDC to Pixel conversion: pix = ((v + 1.0) * S - 1.0) * 0.5
    pix_x = ((means_ndc[:, 0] + 1.0) * image_width - 1.0) * 0.5
    pix_y = ((means_ndc[:, 1] + 1.0) * image_height - 1.0) * 0.5
    means2D = torch.stack([pix_x, pix_y, means_ndc[:, 2]], dim=1)
    
    # Compute 2D Covariance and clamp
    conic_opacity = compute_conic_covariance(means2D, cov3Ds_precomp, scale_modifier)
    
    # Determine tiles touched using tile bounding box calculation
    BLOCK_X = 16 # Example block size
    BLOCK_Y = 16
    max_radius = compute_max_radius(conic_opacity)
    
    rect_min = torch.clamp((means2D[:, :2] - max_radius) / BLOCK_X, min=0).int()
    rect_max = torch.clamp((means2D[:, :2] + max_radius + BLOCK_X - 1) / BLOCK_X, max=(image_width // BLOCK_X)).int()
    
    tiles_touched = (rect_max - rect_min).sum(dim=1)
    
    return means2D, conic_opacity, tiles_touched

def compute_conic_covariance(means2D, cov3D, scale_modifier):
    # Extract 2D submatrix from 3D covariance
    # Apply scale modifier to opacity
    # Return conic matrix and scaled opacity
    pass # Implementation details omitted as per standard graphics knowledge [G]

def compute_max_radius(conic_opacity):
    # Ellipse intersection calculation: h = coord - p_u
    # sqrt_term = sqrt(disc * h^2 + t * coeff)
    # x = (-con_o.y * h - sqrt_term) / coeff + p_v
    # y = (-con_o.y * h + sqrt_term) / coeff + p_v
    # Return max distance
    pass # Implementation details omitted as per standard graphics knowledge [G]
```

### Parameters
*   `mult`: Multiplier for the compact box to control the tile number of each splat [S1, unofficial].
*   `SH_C0`: Spherical Harmonics coefficient constant, value `0.28209479177387814f` [S1, unofficial].
*   `SH_C1`: Spherical Harmonics coefficient constant, value `0.4886025119029199f` [S1, unofficial].
*   `SH_C2`: Spherical Harmonics coefficients array, values `{1.0925484305920792f, -1.0925484305920792f, 0.31539156525252005f, -1.0925484305920792f, 0.5462742152960396f}` [S1, unofficial].
*   `SH_C3`: Spherical Harmonics coefficients array, values `{-0.5900435899266435f, 2.890611442640554f, -0.4570457994644658f, 0.3731763325901154f, -0.4570457994644658f, 1.445305721320277f, -0.5900435899266435f}` [S1, unofficial].
*   `BLOCK_SIZE`: Defined as `BLOCK_X * BLOCK_Y` [S1, unofficial].
*   `NUM_WARPS`: Defined as `BLOCK_SIZE / 32` [S1, unofficial].

### Edge Cases and Pitfalls
*   **Input Validation**: Exactly one of `SHs` or `precomputed colors` must be provided. Exactly one of `scale/rotation` pair or `precomputed 3D covariance` must be provided [S1, unofficial].
*   **Frustum Culling**: Uses a near plane threshold of `0.2f`. Points behind this are discarded [S1, unofficial].
*   **Homogeneous Division**: Near-zero `w` is handled by adding `0.0000001f` to prevent division by zero [S1, unofficial].
*   **Debug Mode**: If `debug` is true, tensors are copied to CPU before CUDA invocation to allow snapshotting on error. Errors save `snapshot_fw.dump` or `snapshot_bw.dump` [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Dominated by the radix sort and bucket-based blending [S1, unofficial].
*   **Memory Complexity**: Requires buffers for sorted keys/values, tile ranges, bucket offsets, and sample states [S1, unofficial].
*   **Performance**: Optimized for high throughput on CUDA-capable GPUs by minimizing memory contention through tile-based binning and radix sorting [S1, unofficial].
*   **Strengths**: High rendering speed, differentiable, supports additional metric buffers [S1, unofficial].
*   **Limitations**: High memory overhead for large scenes due to sorting and bucketing [S1, unofficial].

## Background
The FastGS CUDA Rasterizer is part of the FastGS project, an unofficial extension of 3D Gaussian Splatting [S1, unofficial]. It builds upon the original differentiable Gaussian rasterizer by Inria, adding optimizations for tile-based rendering and additional debugging features [S1, unofficial]. It is used in training and rendering pipelines for 3D Gaussian Splatting [S1, unofficial].

## Sources
[S1] fastgs/FastGS, https://github.com/fastgs/FastGS, fetched 2026-10-27 (unofficial)
