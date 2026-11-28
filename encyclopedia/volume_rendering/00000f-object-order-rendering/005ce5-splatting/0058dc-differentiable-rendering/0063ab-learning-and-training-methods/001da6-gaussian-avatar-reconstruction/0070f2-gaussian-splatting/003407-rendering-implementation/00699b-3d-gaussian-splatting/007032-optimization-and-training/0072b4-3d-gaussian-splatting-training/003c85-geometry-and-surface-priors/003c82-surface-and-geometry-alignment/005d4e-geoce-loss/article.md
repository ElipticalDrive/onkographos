# GeoCE Loss
Also called: Geometric Contrastive Loss

## Overview
GeoCE (Geometric Contrastive Embedding) is a geometric regularization term used in ZipSplat to align token similarity with 3D proximity, ensuring that clustering merges image patches observing the same surface rather than grouping them by viewpoint [S1, unofficial]. It solves the problem where feature-space clustering alone does not guarantee geometric consistency, by explicitly shaping the token space using soft target distributions derived from 3D anchor points [S1, unofficial]. It differs from [Cone Loss](17337-cone-loss.md), which anchors individual tokens to their source patches, by handling the merging of tokens across views based on surface correspondence [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Geometry tokens $\gtok_i \in \mathbb{R}^d$ (one per image patch), ground-truth camera poses, median depth maps for each patch, and a set of sampled anchor tokens.
*   **Outputs:** A scalar loss value $\mathcal{L}_{\mathrm{geo}}$ added to the total training loss.

**Data Structures:**
*   **Geometry Tokens:** $\ell_2$-normalized vectors $\gtok_i$.
*   **3D Anchors:** Points $\spoint_i \in \mathbb{R}^3$ derived from unprojecting patch pixels.
*   **Bandwidths:** Scalars $\sigma_i$ per token.

**Steps:**
1.  **Anchor Sampling:** Sample 2048 tokens per scene as anchors [S1, unofficial].
2.  **3D Unprojection:** For each anchor token $i$, obtain its 3D anchor point $\spoint_i$ by unprojecting the median-depth pixel of its source patch using the ground-truth camera [S1, unofficial].
3.  **Validity Check:** Exclude patches where valid depth exists on fewer than half of their pixels [S1, unofficial].
4.  **Bandwidth Calculation:**
    *   Compute the base bandwidth $\sigma$ as the scene-wise median 3D distance between adjacent patch points [S1, unofficial].
    *   Rank all tokens by their mean image-gradient energy (texture complexity) [S1, unofficial].
    *   Compute per-token bandwidth $\sigma_i$ by log-interpolating between $\sigma$ (for the flattest patch) and $0.4\sigma$ (for the most textured patch) based on the rank [S1, unofficial].
5.  **Target Weight Computation:** Compute the soft target weight $w_{ij}$ for every pair of distinct tokens $(i, j)$ using the Gaussian kernel based on 3D proximity:
    $$ w_{ij} \propto \exp \!\left ( -\frac {\|\spoint _i-\spoint _j\|^2} {2\sigma _i\sigma _j} \right ) $$
    where $\spoint_i$ and $\spoint_j$ are the 3D anchor points, and $\sigma_i, \sigma_j$ are their respective bandwidths [S1, unofficial]. The weight is averaged over a $4 \times 4$ pixel grid inside each key patch [S1, unofficial].
6.  **Loss Calculation:** Compute the GeoCE loss using cross-entropy between the token cosine similarities and the target weights:
    $$ \mathcal{L}_{\mathrm{geo}} = -\sum_i\sum_{j\neq i} w_{ij} \log \frac {\exp (\gtok _i^\top \gtok _j/\tau )} {\sum _{k\neq i}\exp (\gtok _i^\top \gtok _k/\tau )} $$
    where $\tau$ is the temperature parameter [S1, unofficial].

**Parameters:**
*   **Temperature $\tau$:** 0.1 [S1, unofficial].
*   **Loss Weight $\lambda_{\mathrm{geo}}$:** 0.05 [S1, unofficial].
*   **Anchor Count:** 2048 per scene [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_geoce_loss(geometry_tokens, camera_poses, median_depth_maps, patch_masks):
    """
    geometry_tokens: Tensor of shape (N, D), l2-normalized.
    camera_poses: List of N camera pose matrices.
    median_depth_maps: List of N depth maps (H, W).
    patch_masks: List of N boolean masks indicating valid pixels.
    """
    N = len(geometry_tokens)
    anchors = []
    valid_indices = []
    
    # Step 1 & 2: Sample anchors and unproject
    # Note: In practice, 2048 are sampled. Here we assume all N are processed 
    # or a subset is selected. For implementation, select 2048 indices.
    num_anchors = min(2048, N)
    anchor_indices = sample_indices(N, num_anchors)
    
    for idx in anchor_indices:
        # Unproject median depth pixel
        depth_map = median_depth_maps[idx]
        mask = patch_masks[idx]
        
        # Check validity: fewer than half pixels valid -> exclude
        if torch.sum(mask) < (mask.shape[0] * mask.shape[1]) / 2:
            continue
            
        # Get median depth value
        valid_depths = depth_map[mask]
        if len(valid_depths) == 0:
            continue
        median_depth = torch.median(valid_depths)
        
        # Unproject pixel with median depth to 3D point
        # Assuming standard pinhole camera model
        # Find pixel with median depth (or use center if uniform)
        # Simplified: unproject center pixel with median depth
        h, w = depth_map.shape
        u, v = w // 2, h // 2
        
        # Inverse projection
        point_3d = unproject_pixel(camera_poses[idx], u, v, median_depth)
        anchors.append(point_3d)
        valid_indices.append(idx)
        
    if len(anchors) == 0:
        return 0.0
        
    anchors = torch.stack(anchors)
    valid_tokens = geometry_tokens[valid_indices]
    
    # Step 4: Compute bandwidths
    # Compute image gradient energy for ranking
    gradient_energies = []
    for idx in valid_indices:
        # Compute mean gradient magnitude of the patch
        grad_energy = compute_gradient_energy(median_depth_maps[idx], patch_masks[idx])
        gradient_energies.append(grad_energy)
        
    gradient_energies = torch.tensor(gradient_energies)
    ranks = torch.argsort(torch.argsort(gradient_energies)) # Rank 0 is flattest
    
    # Base bandwidth sigma
    # Compute pairwise 3D distances between adjacent patch points (simplified as median dist)
    # In practice, this requires a grid structure. Here we approximate with median dist of anchors.
    dists = torch.cdist(anchors, anchors)
    # Exclude self-distances
    dists = dists.masked_fill(torch.eye(len(anchors), dtype=torch.bool), float('inf'))
    sigma_base = torch.median(dists)
    
    # Log-interpolate sigma_i
    # Flattenest (rank 0) -> sigma_base
    # Most textured (rank max) -> 0.4 * sigma_base
    min_rank = 0
    max_rank = len(ranks) - 1
    if max_rank == 0:
        sigmas = torch.full((len(ranks),), sigma_base)
    else:
        # Log interpolation
        log_sigma_min = torch.log(torch.tensor(sigma_base))
        log_sigma_max = torch.log(torch.tensor(0.4 * sigma_base))
        log_sigma_i = log_sigma_min + (ranks / max_rank) * (log_sigma_max - log_sigma_min)
        sigmas = torch.exp(log_sigma_i)
        
    # Step 5: Compute target weights
    # w_ij proportional to exp(-||p_i - p_j||^2 / (2 * sigma_i * sigma_j))
    dists_sq = torch.cdist(anchors, anchors) ** 2
    sigma_outer = torch.outer(sigmas, sigmas)
    exponent = -dists_sq / (2 * sigma_outer)
    # Mask out self
    exponent = exponent.masked_fill(torch.eye(len(anchors), dtype=torch.bool), float('-inf'))
    weights = torch.softmax(exponent, dim=1) # Normalize to sum to 1 for each i
    
    # Step 6: Compute loss
    # Cosine similarity (tokens are already l2-normalized)
    sim_matrix = valid_tokens @ valid_tokens.T
    tau = 0.1
    logits = sim_matrix / tau
    
    # Cross-entropy with weights
    # log_softmax
    log_probs = torch.nn.functional.log_softmax(logits, dim=1)
    
    # Loss = - sum_i sum_j w_ij log(p_j|i)
    loss = -torch.sum(weights * log_probs)
    
    return loss
```

**Edge Cases and Pitfalls:**
*   **Invalid Depth:** Patches with insufficient valid depth pixels are excluded to prevent noisy 3D anchors [S1, unofficial].
*   **Precision:** The log-interpolation of bandwidths requires careful handling of zero or negative gradient energies, though gradient energy is typically positive [G].
*   **Anchor Count:** Sampling only 2048 anchors limits the scope of the contrastive loss to a subset of the scene, which may miss long-range geometric consistency if not managed [G].

## Subtopics

### [Cone Loss](17337-cone-loss.md)
Anchors individual Gaussians to their source image patches to prevent drifting, handling initial spatial grounding rather than cross-view merging.

## Operating characteristics
*   **Time Complexity:** $O(N^2)$ for the pairwise distance and similarity computations, where $N$ is the number of sampled anchors (2048). This is constant relative to the total number of patches if $N$ is fixed, but linear in the number of anchors if sampled dynamically.
*   **Memory Complexity:** $O(N^2)$ to store the weight matrix and similarity matrix.
*   **Performance:** Contributes to state-of-the-art results on standard benchmarks in ZipSplat by enabling compact scene representations with fewer Gaussians [S1, unofficial].
*   **Strengths:** Explicitly enforces surface-based clustering, reducing viewpoint-dependent artifacts in the token space.
*   **Limitations:** Relies on accurate median depth and ground-truth cameras for anchor generation; sensitive to depth noise [S1, unofficial].

## Background
GeoCE Loss was introduced in the ZipSplat paper (2026) as part of a method for few-view Gaussian splatting [S1, unofficial]. It addresses the limitation of previous clustering methods that did not guarantee geometric consistency across views.

## Sources
[S1] ZipSplat: Fewer Gaussians, Better Splats, https://doi.org/10.48550/arXiv.2606.05102, fetched 2026-10-29 (unofficial)
