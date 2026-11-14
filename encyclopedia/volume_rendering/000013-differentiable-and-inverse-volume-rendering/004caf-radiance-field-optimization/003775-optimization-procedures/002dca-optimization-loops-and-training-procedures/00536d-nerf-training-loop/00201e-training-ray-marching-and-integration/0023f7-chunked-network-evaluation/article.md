# Chunked Network Evaluation

Also called: Batched NeRF Evaluation, Neural Network Chunking

## Overview
Chunked Network Evaluation is a memory-management technique that evaluates a NeRF network on batched points in chunks to avoid out-of-memory errors during training or rendering [S1, unofficial]. It partitions the total set of query points into smaller subsets processed sequentially, rather than attempting to process all points in a single forward pass [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** `pts` (tensor of shape $(N_{rays}, N_{samples}, 3)$), `viewdirs` (tensor of shape $(N_{rays}, 1, 3)$), `model` (the NeRF network function), and `netchunk` (integer specifying the maximum number of points per chunk) [S1, unofficial].
*   **Outputs:** `rgb` (tensor of shape $(N_{rays}, N_{samples}, 3)$) and `sigma` (tensor of shape $(N_{rays}, N_{samples}, 1)$) [S1, unofficial].

**Data Structures:**
*   `all_rgb` and `all_sigma`: Lists of tensors storing intermediate results from each chunk [S1, unofficial].

**Steps:**
1.  **Flatten Inputs:** Reshape `pts` and `viewdirs` into 2D tensors where the first dimension is the total number of points ($N_{total} = N_{rays} \times N_{samples}$) [S1, unofficial].
    $$ pts_{flat} = pts.reshape(-1, 3) $$
    $$ viewdirs_{flat} = viewdirs[:, None, :].expand_as(pts).reshape(-1, 3) $$
2.  **Initialize Storage:** Create empty lists `all_rgb` and `all_sigma` to store results from each chunk [S1, unofficial].
3.  **Iterate and Evaluate:** Loop through the flattened points in steps of `netchunk` [S1, unofficial]. For each chunk index $i$:
    *   Extract `chunk_pts` from `pts_flat[i : i + netchunk]` and `chunk_dirs` from `viewdirs_flat[i : i + netchunk]` [S1, unofficial].
    *   Call `model(chunk_pts, chunk_dirs)` to get `chunk_rgb` and `chunk_sigma` [S1, unofficial].
    *   Append `chunk_rgb` to `all_rgb` and `chunk_sigma` to `all_sigma` [S1, unofficial].
4.  **Concatenate and Reshape:** Concatenate all lists along dimension 0 and reshape back to the original 3D structure [S1, unofficial].
    $$ rgb = concatenate(all\_rgb, dim=0).reshape(N_{rays}, N_{samples}, 3) $$
    $$ sigma = concatenate(all\_sigma, dim=0).reshape(N_{rays}, N_{samples}, 1) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def chunked_network_evaluation(pts, viewdirs, model, netchunk):
    pts_flat = pts.reshape(-1, 3)
    viewdirs_flat = viewdirs[:, None, :].expand_as(pts).reshape(-1, 3)
    N_total = pts_flat.shape[0]
    all_rgb = []
    all_sigma = []
    for i in range(0, N_total, netchunk):
        chunk_pts = pts_flat[i : i + netchunk]
        chunk_dirs = viewdirs_flat[i : i + netchunk]
        chunk_rgb, chunk_sigma = model(chunk_pts, chunk_dirs)
        all_rgb.append(chunk_rgb)
        all_sigma.append(chunk_sigma)
    rgb = concatenate(all_rgb, dim=0).reshape(pts.shape[0], pts.shape[1], 3)
    sigma = concatenate(all_sigma, dim=0).reshape(pts.shape[0], pts.shape[1], 1)
    return rgb, sigma

def concatenate(tensors, dim=0):
    # Standard tensor concatenation along dimension dim [G]
    return tensors[0] if len(tensors) == 1 else tensors[0] + sum(tensors[1:], tensors[0])
```
[S1, unofficial]

**Parameters:**
*   `netchunk`: Not given in the sources.

**Edge Cases and Pitfalls:**
*   Not given in the sources.

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

## Operating characteristics
*   **Time Complexity:** $O(N_{rays} \times N_{samples})$ [S1, unofficial].
*   **Memory Complexity:** $O(netchunk)$ for the network inputs and outputs, instead of $O(N_{rays} \times N_{samples})$ [S1, unofficial].
*   **Strengths:** Enables processing large point sets without exceeding memory limits [S1, unofficial].
*   **Limitations:** Sequential processing introduces loop overhead compared to a single forward pass [S1, unofficial].
*   **When to use:** When the full batch of points exceeds available memory [S1, unofficial].

## Background
Chunked Network Evaluation appears in open-source NeRF implementations and tutorials, such as "NeRF from Scratch" [S1, unofficial]. The technique is also adapted in variants like Point-NeRF, where it is used to manage memory during density queries with pre-computed features [S2, unofficial].

## Sources
[S1] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S2] Xharlie/pointnerf, https://github.com/Xharlie/pointnerf, fetched 2026-10-26 (unofficial)
