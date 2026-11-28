# Error-Guided Sparse Pixel Sampling

Also called: Err-SPS, Tile-Wise Pixel Sampling (EPS), Tile-wise Sparse Pixel Sampling, sparse pixel training

## Overview
Error-Guided Sparse Pixel Sampling (Err-SPS) is a training acceleration strategy for 3D Gaussian Splatting that restricts loss evaluation and gradient computation to a sparse subset of pixels per tile, guided by persistent error and age maps [S1, unofficial]. It solves the problem of redundant gradient computation in well-reconstructed regions by reallocating computation to perceptually informative pixels [S1, unofficial]. It is distinguished from sibling methods like [TurboGS](15697-turbogs.md) by its specific focus on the sampling mechanism itself, whereas TurboGS integrates Err-SPS with additional components like structure-aware loss and density control [S1, unofficial]. Unlike [Frequency Guided Resolution Scheduler](15751-frequency-guided-resolution-scheduler.md) which targets resolution, Err-SPS targets pixel-level supervision density [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current Gaussian set $\mathcal{G}$, ground-truth images $I_{gt}$, camera poses $C_v$, tile configuration (tile size, grid dimensions), error map $E$, age map $A$, training progress $\rho_t$, and sampling rate parameters.
*   **Outputs:** Updated Gaussian parameters $\mathcal{G}'$, updated error map $E'$, updated age map $A'$, and sparse gradients for the optimizer.

**Data Structures:**
*   **Error Map ($E$):** A 2D array matching the image resolution (or tile grid), storing the accumulated reconstruction error per pixel. Initialized to zero or a small constant.
*   **Age Map ($A$):** A 2D array matching the image resolution, storing the number of consecutive iterations a pixel has *not* been sampled. Initialized to zero.
*   **Sampled Pixel Set ($P_{t,v}$):** A list of pixel coordinates $(u, v)$ for view $v$ in tile $t$ selected for the current iteration.

**Numbered Steps:**
1.  **Initialize:** Set $E = 0$, $A = 0$. Define tile size $T \times T$ and target sampling ratio $r_t$ (e.g., 10-20% of pixels).
2.  **Iterate:** For each training iteration $t$:
    3.  **View Selection:** Select a set of views $V_t$ for the current tile batch. [S1, unofficial] Note: Err-SPS alone does not define view selection; it assumes views are provided. In TurboGS, this is geometry-aware [S1, unofficial].
    4.  **Tile-wise Sampling:** For each tile $t$ and view $v \in V_t$:
        5.  **Query Error:** Retrieve error values $E(p)$ for all pixels $p$ in the tile.
        6.  **Sort:** Sort pixels in the tile by error $E(p)$ in descending order.
        7.  **Select:** Select the top $k = \lfloor r_t \times T^2 \rfloor$ pixels as the sampled set $P_{t,v}$.
        8.  **Increment Age:** For all pixels $p$ in the tile *not* in $P_{t,v}$, increment $A(p) \leftarrow A(p) + 1$.
        9.  **Reset Age:** For all pixels $p \in P_{t,v}$, reset $A(p) \leftarrow 0$.
    10. **Forward Pass:** Render the scene using the selected pixels $P_{t,v}$ for views $V_t$. This produces rendered colors $\hat{I}$ and depths $\hat{D}$ only for sampled pixels.
    11. **Loss Computation:** Compute the loss $L$ only over sampled pixels:
        $$ L = \sum_{v \in V_t} \sum_{p \in P_{t,v}} \ell_1(\hat{I}_p, I_{gt,p}) + \lambda_\tau^{ncc} L_\tau^{S-NCC} $$
        where $\ell_1$ is the L1 loss, and $L_\tau^{S-NCC}$ is a sparse Normalized Cross-Correlation term [S1, unofficial].
    12. **Backward Pass:** Compute gradients $\nabla \mathcal{G}$ with respect to the Gaussian parameters using the sparse loss. The backward pass is optimized via a per-Gaussian mechanism to handle sparse pixel updates efficiently [S1, unofficial].
    13. **Update Parameters:** Update $\mathcal{G}$ using the Adam optimizer with the computed gradients [S1, unofficial].
    14. **Update Error Map:** Update $E$ using an Exponential Moving Average (EMA) of the current pixel-wise losses:
        $$ E(p) \leftarrow \alpha E(p) + (1 - \alpha) \ell_1(\hat{I}_p, I_{gt,p}) \quad \forall p \in P_{t,v} $$
        For unsampled pixels, $E(p)$ remains unchanged [S1, unofficial].
    15. **Update Age Map:** (Already handled in step 9).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def error_guided_sparse_pixel_sampling(Gaussians, GT_Images, Cameras, Error_Map, Age_Map, Iteration, Sampling_Rate, Alpha_EMA):
    # 1. View Selection (Assumed external or random for pure Err-SPS)
    Views = select_views(Cameras) 
    
    # 2. Tile-wise Sampling
    Sampled_Pixels = {} # Dict: (view_idx, tile_idx) -> list of (u, v)
    
    for view_idx, camera in enumerate(Views):
        # Get tile grid for this view
        Tiles = get_tile_grid(camera.image_width, camera.image_height)
        
        for tile_idx, tile in enumerate(Tiles):
            tile_pixels = get_pixels_in_tile(tile)
            
            # Get errors for current tile pixels
            errors = [Error_Map[p.y][p.x] for p in tile_pixels]
            
            # Sort pixels by error descending
            sorted_indices = argsort_desc(errors)
            
            # Select top k pixels
            k = int(Sampling_Rate * len(tile_pixels))
            selected_indices = sorted_indices[:k]
            sampled_set = [tile_pixels[i] for i in selected_indices]
            
            Sampled_Pixels[(view_idx, tile_idx)] = sampled_set
            
            # Update Age Map
            for p in tile_pixels:
                if p not in sampled_set:
                    Age_Map[p.y][p.x] += 1
                else:
                    Age_Map[p.y][p.x] = 0

    # 3. Forward Pass (Sparse Rasterization)
    Rendered_Images = {}
    Rendered_Depths = {}
    
    for (view_idx, tile_idx), pixels in Sampled_Pixels.items():
        camera = Views[view_idx]
        # Rasterize only for these pixels
        rendered_color, rendered_depth = sparse_rasterize(Gaussians, camera, pixels)
        Rendered_Images[(view_idx, tile_idx)] = rendered_color
        Rendered_Depths[(view_idx, tile_idx)] = rendered_depth

    # 4. Loss Computation
    Total_Loss = 0
    Pixel_Losses = {} # Store losses for EMA update
    
    for (view_idx, tile_idx), pixels in Sampled_Pixels.items():
        gt_color = GT_Images[view_idx]
        rendered_color = Rendered_Images[(view_idx, tile_idx)]
        
        # Compute L1 loss for sampled pixels
        loss = 0
        for p in pixels:
            l1 = abs(rendered_color[p.y][p.x] - gt_color[p.y][p.x])
            loss += l1
            Pixel_Losses[(view_idx, tile_idx, p)] = l1
            
        Total_Loss += loss
        
        # Add Sparse NCC if required (TurboGS component)
        # loss += lambda_ncc * sparse_ncc_loss(...)

    # 5. Backward Pass
    Gradients = compute_gradients(Total_Loss, Gaussians)
    
    # 6. Update Gaussians
    Gaussians = adam_update(Gaussians, Gradients)
    
    # 7. Update Error Map (EMA)
    for (view_idx, tile_idx), pixels in Sampled_Pixels.items():
        for p in pixels:
            current_loss = Pixel_Losses[(view_idx, tile_idx, p)]
            Error_Map[p.y][p.x] = Alpha_EMA * Error_Map[p.y][p.x] + (1 - Alpha_EMA) * current_loss
            
    return Gaussians, Error_Map, Age_Map, Total_Loss
```

**Parameters:**
*   **Sampling Rate ($r_t$):** The fraction of pixels to sample per tile. Typical values are 10-20% [S1, unofficial].
*   **EMA Alpha ($\alpha$):** The smoothing factor for the error map. Not explicitly given in sources, but standard EMA values (e.g., 0.9) are implied [S1, unofficial].
*   **$\lambda_\tau^{ncc}$:** Weight for sparse NCC loss. Not given in sources [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Degenerate Error Maps:** If the error map is not initialized properly, early training may sample randomly or bias towards specific regions [S1, unofficial].
*   **Stale Errors:** If the error map is not updated frequently enough, it may guide sampling to regions that are no longer challenging [S1, unofficial].
*   **Loss of High-Frequency Details:** Relying solely on sparse L1 supervision can degrade reconstruction quality, particularly in high-frequency regions [S1, unofficial]. This requires complementary techniques like sparse NCC regularization or geometry-aware view sampling [S1, unofficial].
*   **Rippling Artifacts:** Sparse supervision may introduce subtle rippling artifacts in low-frequency regions with weak texture variations (e.g., walls) [S1, unofficial].

## Subtopics

### TurboGS
A full training framework that incorporates Err-SPS along with structure-aware loss, density control, and a hybrid optimizer. Choose when you need end-to-end acceleration with quality preservation. [TurboGS](15697-turbogs.md)

### Frequency Guided Resolution Scheduler
Adjusts resolution based on frequency content. Choose when you want to manage computational load via resolution rather than pixel density. [Frequency Guided Resolution Scheduler](15751-frequency-guided-resolution-scheduler.md)

### Sparse SH Order Increment
Adjusts SH order sparsely. Choose when optimizing appearance details rather than geometry/speed. [Sparse SH Order Increment](18689-sparse-sh-order-increment.md)

## Operating characteristics
*   **Time Complexity:** Reduces training time significantly. On Mip-NeRF 360, Err-SPS reduces training time from 891s to 85s [S1, unofficial].
*   **Memory Complexity:** Reduces the number of Gaussian splats (NGS) from 2.73M to 0.76M during training, indicating lower memory pressure for gradient accumulation and parameter updates [S1, unofficial].
*   **Performance:** Provides substantial acceleration with minimal quality loss when combined with complementary techniques (NCC, density control) [S1, unofficial].
*   **Strengths:** Efficiently reallocates computation to informative pixels; avoids redundant gradient computation in well-reconstructed regions.
*   **Limitations:** Can degrade reconstruction quality if used alone; may introduce artifacts in low-texture regions; requires careful tuning of sampling rate and error map update frequency [S1, unofficial].
*   **When to Use:** When training time is a bottleneck and you can afford to integrate complementary loss terms (like NCC) to maintain quality.
*   **When Not to Use:** When high-fidelity reconstruction of fine details is critical and you cannot use additional regularization; or when hardware memory is not the primary constraint.

## Background
Err-SPS was introduced in the TurboGS paper by Hanson et al. in 2026 (preprint) [S1, unofficial]. It is part of the broader effort to accelerate 3D Gaussian Splatting training, building on the observation that most pixels become well-reconstructed early in training, yielding marginal gradients [S1, unofficial].

## Sources
[S1] TurboGS: Accelerating 3D Gaussian Splatting via Error-Guided Sparse Pixel Sampling and Optimization, https://doi.org/10.48550/arXiv.2606.15924, fetched 2026-10-29 (unofficial)
