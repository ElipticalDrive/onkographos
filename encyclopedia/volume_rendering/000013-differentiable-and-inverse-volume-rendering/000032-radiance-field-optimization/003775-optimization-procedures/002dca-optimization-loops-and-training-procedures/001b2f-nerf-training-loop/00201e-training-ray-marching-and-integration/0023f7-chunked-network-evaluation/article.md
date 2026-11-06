# Chunked Network Evaluation

Also called: Batched NeRF Evaluation, Neural Network Chunking

## Overview
Chunked Network Evaluation is a memory-management technique used during the training and rendering of Neural Radiance Fields (NeRF) to avoid out-of-memory (OOM) errors when evaluating the neural network on large batches of 3D points [S1, unofficial]. It solves the problem where the number of samples per ray ($N_{samples}$) multiplied by the number of rays ($N_{rays}$) exceeds the GPU's VRAM capacity for the network's forward pass. It sets itself apart from standard batched evaluation by explicitly partitioning the point cloud into smaller, manageable subsets (chunks) processed sequentially, rather than attempting to process all points in a single tensor operation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `pts`: Tensor of shape $(N_{rays}, N_{samples}, 3)$ containing 3D coordinates.
    *   `viewdirs`: Tensor of shape $(N_{rays}, 1, 3)$ containing view directions for each ray.
    *   `model`: The NeRF neural network function.
    *   `netchunk`: Integer specifying the maximum number of points to process per chunk (e.g., 4096 or 8192) [S1, unofficial].
*   **Outputs:**
    *   `rgb`: Tensor of shape $(N_{rays}, N_{samples}, 3)$ containing predicted colors.
    *   `sigma`: Tensor of shape $(N_{rays}, N_{samples}, 1)$ containing predicted volume densities.

**Data Structures:**
*   `all_rgb`: List of tensors, each of shape $(netchunk, 3)$.
*   `all_sigma`: List of tensors, each of shape $(netchunk, 1)$.

**Steps:**
1.  **Flatten Inputs:** Reshape `pts` and `viewdirs` into 2D tensors where the first dimension is the total number of points ($N_{total} = N_{rays} \times N_{samples}$).
    $$ pts_{flat} = pts.reshape(-1, 3) $$
    $$ viewdirs_{flat} = viewdirs[:, None, :].expand_as(pts).reshape(-1, 3) $$
    [S1, unofficial]
2.  **Initialize Storage:** Create empty lists `all_rgb` and `all_sigma` to store results from each chunk.
3.  **Iterate and Evaluate:** Loop through the flattened points in steps of `netchunk`. For each chunk index $i$:
    *   Extract `chunk_pts` from `pts_flat[i : i + netchunk]`.
    *   Extract `chunk_dirs` from `viewdirs_flat[i : i + netchunk]`.
    *   Call `model(chunk_pts, chunk_dirs)` to get `chunk_rgb` and `chunk_sigma`.
    *   Append `chunk_rgb` to `all_rgb` and `chunk_sigma` to `all_sigma`.
    [S1, unofficial]
4.  **Concatenate and Reshape:** Concatenate all lists along dimension 0 and reshape back to the original 3D structure.
    $$ rgb = cat(all\_rgb, dim=0).reshape(N_{rays}, N_{samples}, 3) $$
    $$ sigma = cat(all\_sigma, dim=0).reshape(N_{rays}, N_{samples}, 1) $$
    [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def chunked_network_evaluation(pts, viewdirs, model, netchunk):
    """
    Evaluates NeRF network on points in chunks to manage memory.
    
    Args:
        pts: Tensor (N_rays, N_samples, 3)
        viewdirs: Tensor (N_rays, 1, 3)
        model: Callable taking (points, viewdirs) -> (rgb, sigma)
        netchunk: int, number of points per chunk
        
    Returns:
        rgb: Tensor (N_rays, N_samples, 3)
        sigma: Tensor (N_rays, N_samples, 1)
    """
    # Flatten inputs
    pts_flat = pts.reshape(-1, 3)
    # Expand viewdirs to match pts shape before flattening
    viewdirs_flat = viewdirs[:, None, :].expand_as(pts).reshape(-1, 3)
    
    N_total = pts_flat.shape[0]
    all_rgb = []
    all_sigma = []
    
    # Iterate in chunks
    for i in range(0, N_total, netchunk):
        chunk_pts = pts_flat[i : i + netchunk]
        chunk_dirs = viewdirs_flat[i : i + netchunk]
        
        # Evaluate model on chunk
        chunk_rgb, chunk_sigma = model(chunk_pts, chunk_dirs)
        
        all_rgb.append(chunk_rgb)
        all_sigma.append(chunk_sigma)
        
    # Concatenate results
    rgb = torch.cat(all_rgb, dim=0).reshape(pts.shape[0], pts.shape[1], 3)
    sigma = torch.cat(all_sigma, dim=0).reshape(pts.shape[0], pts.shape[1], 1)
    
    return rgb, sigma
```
[S1, unofficial]

**Parameters:**
*   `netchunk`: Typical values are 4096 or 8192, depending on GPU VRAM and network size [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Concatenation and reshaping operations are standard and do not introduce significant precision loss, but ensure that `netchunk` divides the total number of points evenly or handle the final partial chunk correctly (the loop range handles this naturally).
*   **Boundaries:** The loop `range(0, N_total, netchunk)` correctly handles cases where $N_{total}$ is not a multiple of `netchunk`.
*   **Memory:** If `netchunk` is too small, overhead from repeated function calls may increase training time. If too large, OOM errors persist.

## Subtopics

### Point-NeRF Density Query
A variant used in Point-NeRF to compute volume density for rays by querying a neural network with ray points and pre-computed density features, processed in chunks to manage memory [S2, unofficial]. It differs from standard chunked evaluation by incorporating pre-computed features (`density_feature`) into the query function [S2, unofficial].
*   **Definition:** Computes density for rays in a Point-NeRF by querying a neural network with ray points and pre-computed density features, processed in chunks [S2, unofficial].
*   **Steps:**
    1.  Initialize an empty list `densities` and determine the device from `density_feature` [S2, unofficial].
    2.  Iterate over `rays_pts` in chunks of size `chunk` (default 5120) [S2, unofficial].
    3.  For each chunk index $i$, extract `input_feat = rays_pts[i:i + chunk]` and move it to the device [S2, unofficial].
    4.  Call `network_query_fn` with `input_feat`, `None`, `density_feature[i:i + chunk]`, and `network_fn` to get the density for the current chunk, and append it to `densities` [S2, unofficial].
    5.  Concatenate all density tensors in the `densities` list along the default dimension and return the result [S2, unofficial].
*   **When to choose:** Use when working with Point-NeRF architectures that rely on pre-computed density features rather than raw coordinates for density prediction [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N_{rays} \times N_{samples})$, same as unchunked evaluation, but with higher constant factors due to loop overhead and multiple kernel launches [S1, unofficial].
*   **Memory Complexity:** $O(netchunk)$ for the network inputs and outputs, instead of $O(N_{rays} \times N_{samples})$ [S1, unofficial].
*   **Measured Performance:** Performance depends heavily on `netchunk`. Too small leads to CPU-GPU synchronization overhead; too large leads to OOM [S1, unofficial].
*   **Strengths:** Enables training on large scenes or high-resolution rays that would otherwise exceed VRAM limits [S1, unofficial].
*   **Limitations:** Slightly slower than a single batched forward pass due to loop overhead [S1, unofficial].
*   **When to use:** When VRAM is insufficient for the full batch of points [S1, unofficial].
*   **When not to use:** When VRAM is sufficient, as a single batched pass is more efficient [S1, unofficial].

## Background
Chunked Network Evaluation is a practical implementation detail arising from the memory constraints of GPU hardware during the training of NeRF models. It is commonly found in open-source NeRF implementations and tutorials, such as "NeRF from Scratch" [S1, unofficial]. The technique is also adapted in variants like Point-NeRF, where it is used to manage memory during density queries with pre-computed features [S2, unofficial].

## Sources
[S1] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S2] Xharlie/pointnerf, https://github.com/Xharlie/pointnerf, fetched 2026-10-26 (unofficial)
