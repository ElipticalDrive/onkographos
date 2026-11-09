# Voxel-Aware Masked Ray Sampling

Also called: NSVF Hierarchical Ray Sampling with Voxel-Aware Masking, NSVF parallel ray sampling, NSVF voxel intersection sampling

## Overview
Voxel-Aware Masked Ray Sampling is a ray sampling strategy for sparse voxel fields that samples points only within intersected voxels, handling variable numbers of intersections per ray by padding and masking, and supporting hierarchical sampling ratios [S1, unofficial]. It differs from standard NeRF uniform or stratified sampling by using voxel-based intersection sampling with probabilities proportional to distance [S1, unofficial]. It builds on [Uniform Ray Marching](4810-uniform-ray-marching.md) by replacing fixed intervals with adaptive steps derived from voxel geometry.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origins, ray directions, voxel grid structure (implicit in intersection logic), `chunk_size` (default 4096), `max_hits` (default 60), `raymarching_stepsize` (default 0.01), `fine_num_sample_ratio` (default 0), `inverse_distance_coarse_sampling` (default 'none'), `background_depth` (default 5.0).
*   **Outputs:** Sampled depths (`sampled_dists`), sampled voxel indices (`sampled_idx`), sampled depths relative to voxel boundaries (`sampled_depth`), and compositing results (colors, depths, missed masks).

**Data Structures:**
*   `intersection_outputs`: Dictionary containing:
    *   `min_depth`: Tensor of minimum depths per ray.
    *   `max_depth`: Tensor of maximum depths per ray.
    *   `intersected_voxel_idx`: Tensor of intersected voxel indices.
    *   `probs`: Tensor of sampling probabilities.
    *   `steps`: Tensor of number of steps per ray.
*   `all_results`: Dictionary in postprocessing containing:
    *   `missed`: Tensor of size `(fullsize,)` indicating missed rays.
    *   `colors`: Tensor of size `(fullsize, 3)`.
    *   `depths`: Tensor of size `(fullsize,)`.
    *   `normal`: Tensor of size `(fullsize, 3)` (optional).
    *   `voxel_depth`: Tensor of size `(fullsize,)` (optional).
    *   `voxel_edges`: Tensor of size `(fullsize, 3)` (optional).
    *   `feat_n2`: Tensor of size `(fullsize,)` (optional).

**Numbered Steps:**

1.  **Parallel Chunking:** If the total number of rays `full_size` exceeds `chunk_size` (4096), split the input rays into chunks of size `chunk_size`. Process each chunk independently using the core sampling logic, then zip the results [S1, unofficial].
2.  **Padding:** Find the maximum length `maxlen` among all processed chunks. Create a result tensor of shape `(full_size, maxlen)` filled with padding values. Copy each chunk's results into the corresponding row range [S1, unofficial].
3.  **Mask Identification:** Identify rays that hit the maximum depth boundary by comparing `sampled_depth` to `MAX_DEPTH`. Create a boolean mask: `sampled_masks = (sampled_depth == MAX_DEPTH)` [S1, unofficial].
4.  **Step Count Calculation:** Calculate the maximum number of valid steps across all rays by counting non-masked entries: `num_max_steps = (~sampled_masks).sum(-1).max()` [S1, unofficial].
5.  **Truncation:** Truncate `sampled_depth` to `num_max_steps`: `sampled_depth = sampled_depth[:, :num_max_steps]` [S1, unofficial].
6.  **Gathering and Masking:**
    *   Gather `sampled_dists` using `ordered_index`. Fill masked positions with `0.0`. Truncate to `num_max_steps`: `sampled_dists = sampled_dists.gather(1, ordered_index).masked_fill_(sampled_masks, 0.0)[:, :num_max_steps]` [S1, unofficial].
    *   Gather `sampled_idx` using `ordered_index`. Fill masked positions with `-1`. Truncate to `num_max_steps`: `sampled_idx = sampled_idx.gather(1, ordered_index).masked_fill_(sampled_masks, -1)[:, :num_max_steps]` [S1, unofficial].
7.  **Distance Calculation:** Calculate distances between `min_depth` and `max_depth`. Mask out invalid voxels (where `pts_idx == -1`) by setting their distance to `0`: `dists = (max_depth - min_depth).masked_fill(pts_idx.eq(-1), 0)` [S1, unofficial].
8.  **Probability Normalization:** Compute probabilities for sampling steps by normalizing `dists` by their sum along the last dimension: `probs = dists / dists.sum(dim=-1, keepdim=True)` [S1, unofficial].
9.  **Step Count Determination:**
    *   If `fixed_num_samples > 0`, set `steps` to a constant tensor of value `self.args.fixed_num_samples` [S1, unofficial].
    *   Otherwise, set `steps = dists.sum(-1) / self.encoder.step_size` [S1, unofficial].
10. **Hierarchical Sampling (Optional):** In `prepare_hierarchical_sampling`, if `fine_num_sample_ratio > 0`, update `steps` to be the count of valid voxel indices multiplied by the ratio: `steps = samples['sampled_point_voxel_idx'].ne(-1).sum(-1).float() * self.args.fine_num_sample_ratio` [S1, unofficial].
11. **Postprocessing Background Fill:**
    *   Identify missed rays: `missed = fill_in((fullsize,), hits, all_results['missed'], 1.0)` [S1, unofficial].
    *   Fill missed rays with background values:
        *   `colors = fill_in((fullsize, 3), hits, all_results['colors'], 0.0)` [S1, unofficial].
        *   `depths = fill_in((fullsize,), hits, all_results['depths'], 0.0)` [S1, unofficial].
    *   Add background contribution:
        *   `colors += missed.unsqueeze(-1) * bg_color` [S1, unofficial].
        *   `depths += missed * BG_DEPTH`, where `bg_color` is derived from `all_results['colors']` and `BG_DEPTH` is `self.field.bg_color.depth` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def voxel_aware_masked_ray_sampling(
    rays, 
    chunk_size=4096, 
    max_hits=60, 
    raymarching_stepsize=0.01, 
    fine_num_sample_ratio=0.0, 
    fixed_num_samples=-1,
    background_depth=5.0
):
    """
    Simulates the NSVF voxel-aware masked ray sampling logic.
    Assumes helper functions for intersection and compositing are provided.
    """
    full_size = rays.shape[0]
    
    # Step 1 & 2: Chunking and Padding
    if full_size > chunk_size:
        chunks = torch.split(rays, chunk_size, dim=0)
        results = []
        for chunk in chunks:
            # Process chunk (simplified call)
            chunk_res = _process_chunk(chunk, max_hits, raymarching_stepsize, 
                                       fine_num_sample_ratio, fixed_num_samples, 
                                       background_depth)
            results.append(chunk_res)
        
        # Pad results
        max_len = max(r['sampled_dists'].shape[1] for r in results)
        padded_results = []
        for r in results:
            padded = _pad_to_length(r, max_len)
            padded_results.append(padded)
        
        # Zip results (stack along batch dim)
        final_sampled_dists = torch.cat([r['sampled_dists'] for r in padded_results], dim=0)
        final_sampled_idx = torch.cat([r['sampled_idx'] for r in padded_results], dim=0)
        final_sampled_depth = torch.cat([r['sampled_depth'] for r in padded_results], dim=0)
        final_missed = torch.cat([r['missed'] for r in padded_results], dim=0)
        final_colors = torch.cat([r['colors'] for r in padded_results], dim=0)
        final_depths = torch.cat([r['depths'] for r in padded_results], dim=0)
    else:
        final_sampled_dists, final_sampled_idx, final_sampled_depth, final_missed, final_colors, final_depths = \
            _process_chunk(rays, max_hits, raymarching_stepsize, fine_num_sample_ratio, fixed_num_samples, background_depth)

    return {
        'sampled_dists': final_sampled_dists,
        'sampled_idx': final_sampled_idx,
        'sampled_depth': final_sampled_depth,
        'missed': final_missed,
        'colors': final_colors,
        'depths': final_depths
    }

def _process_chunk(rays, max_hits, raymarching_stepsize, fine_num_sample_ratio, fixed_num_samples, background_depth):
    # Step 3: Mask Identification
    # Assume intersection logic returns sampled_depth
    sampled_depth = _get_sampled_depth(rays, max_hits) 
    sampled_masks = (sampled_depth == float('inf')) # Assuming MAX_DEPTH is inf or large value
    
    # Step 4: Step Count Calculation
    num_max_steps = (~sampled_masks).sum(dim=1).max().item()
    
    # Step 5: Truncation
    sampled_depth = sampled_depth[:, :num_max_steps]
    
    # Step 6: Gathering and Masking
    # Assume ordered_index is generated from intersection logic
    ordered_index = _get_ordered_index(rays, max_hits)
    
    sampled_dists = _get_sampled_dists(rays, max_hits)
    sampled_dists = sampled_dists.gather(1, ordered_index).masked_fill_(sampled_masks, 0.0)[:, :num_max_steps]
    
    sampled_idx = _get_sampled_idx(rays, max_hits)
    sampled_idx = sampled_idx.gather(1, ordered_index).masked_fill_(sampled_masks, -1)[:, :num_max_steps]
    
    # Step 7: Distance Calculation
    min_depth = _get_min_depth(rays)
    max_depth = _get_max_depth(rays)
    pts_idx = sampled_idx
    
    dists = (max_depth - min_depth).masked_fill(pts_idx.eq(-1), 0)
    
    # Step 8: Probability Normalization
    probs = dists / dists.sum(dim=-1, keepdim=True)
    
    # Step 9: Step Count Determination
    if fixed_num_samples > 0:
        steps = torch.full_like(dists[:, 0:1], fixed_num_samples)
    else:
        steps = dists.sum(-1) / raymarching_stepsize
    
    # Step 10: Hierarchical Sampling (Optional)
    if fine_num_sample_ratio > 0:
        # Assume samples['sampled_point_voxel_idx'] is available
        samples_voxel_idx = sampled_idx
        steps = samples_voxel_idx.ne(-1).sum(-1).float() * fine_num_sample_ratio
    
    # Step 11: Postprocessing Background Fill
    # Assume compositing results in all_results
    all_results = _composite(rays, sampled_dists, sampled_idx, dists, probs, steps)
    
    missed = _fill_in((rays.shape[0],), all_results['hits'], all_results['missed'], 1.0)
    colors = _fill_in((rays.shape[0], 3), all_results['hits'], all_results['colors'], 0.0)
    depths = _fill_in((rays.shape[0],), all_results['hits'], all_results['depths'], 0.0)
    
    bg_color = all_results['colors'].mean(dim=0) # Placeholder for bg_color derivation
    bg_depth = background_depth
    
    colors = colors + missed.unsqueeze(-1) * bg_color
    depths = depths + missed * bg_depth
    
    return {
        'sampled_dists': sampled_dists,
        'sampled_idx': sampled_idx,
        'sampled_depth': sampled_depth,
        'missed': missed,
        'colors': colors,
        'depths': depths
    }

def _pad_to_length(result_dict, max_len):
    # Pad tensors to max_len along dim 1
    for key in ['sampled_dists', 'sampled_idx', 'sampled_depth']:
        if key in result_dict:
            tensor = result_dict[key]
            pad_size = max_len - tensor.shape[1]
            if pad_size > 0:
                if key == 'sampled_idx':
                    pad_value = -1
                elif key == 'sampled_depth':
                    pad_value = float('inf')
                else:
                    pad_value = 0.0
                result_dict[key] = torch.nn.functional.pad(tensor, (0, pad_size), value=pad_value)
    return result_dict

def _fill_in(shape, hits, data, fill_value):
    # Fill missed rays with fill_value
    # hits is a boolean mask of hit rays
    if data is None:
        return torch.zeros(shape, dtype=torch.float32)
    # Ensure shape matches
    if data.shape != shape:
        # Resize or broadcast as needed
        pass
    filled = torch.full(shape, fill_value, dtype=torch.float32)
    filled[hits] = data[hits]
    return filled
```

**Parameters:**
*   `chunk_size`: The number of rays processed in parallel before padding. Default: 4096 [S1, unofficial].
*   `fine_num_sample_ratio`: Ratio of samples for the fine pass compared to the coarse pass. Default: 0 [S1, unofficial].
*   `inverse_distance_coarse_sampling`: Controls non-uniform sampling through voxels. Choices: 'none', 'camera', 'origin'. Default: 'none' [S1, unofficial].
*   `max_hits`: Maximum number of voxel intersections per ray. Default: 60 [S1, unofficial].
*   `raymarching_stepsize`: Base step size for raymarching. Default: 0.01 [S1, unofficial].
*   `background_depth`: Depth value for background. Default: 5.0 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   Rays that do not hit any voxel (missed) must be filled with background color and depth to prevent NaNs or incorrect compositing [S1, unofficial].
*   Invalid voxel indices (`pts_idx == -1`) must be masked out when calculating distances to avoid incorrect step counts [S1, unofficial].
*   Precision issues may arise when `sampled_depth` equals `MAX_DEPTH` if not handled with a robust equality check or tolerance [G].

## Operating characteristics
*   **Time Complexity:** Depends on the number of voxel intersections per ray. Padding and masking operations are $O(N \cdot K)$ where $N$ is the number of rays and $K$ is the maximum hits.
*   **Memory Complexity:** Requires storing tensors of size $(N, K)$ for depths, indices, and distances. Padding increases memory usage if rays have varying intersection counts [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales with the number of rays via chunking. Performance degrades if `max_hits` is set too high for sparse scenes due to padding overhead [S1, unofficial].
*   **Strengths:** Adaptive sampling based on voxel geometry; handles sparse fields efficiently by ignoring empty space.
*   **Limitations:** Requires voxel intersection logic; padding overhead for rays with varying intersection counts; complex masking logic [S1, unofficial].
*   **When to use:** Sparse voxel fields where uniform sampling is inefficient.
*   **When not to use:** Dense fields or when hardware does not support efficient tensor masking/padding.

## Background
Voxel-Aware Masked Ray Sampling is part of the NSVF (Neural Sparse Voxel Fields) framework developed by Facebook Research [S1, unofficial]. It was introduced to optimize ray sampling for sparse neural fields, reducing computation by focusing on intersected voxels [S1, unofficial].

## Sources
[S1] facebookresearch/NSVF, https://github.com/facebookresearch/NSVF, fetched 2026-10-27 (unofficial)
