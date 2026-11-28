# Tile-based Gaussian Rasterizer

Also called: Fast Differentiable Rasterizer for Gaussians

## Overview
The tile-based Gaussian rasterizer is a GPU-accelerated rendering method for 3D Gaussian splats that pre-sorts primitives across the entire image to enable approximate alpha-blending for anisotropic splats [S1, unofficial]. It distinguishes itself from previous per-pixel sorting approaches by eliminating hard limits on the number of blended primitives receiving gradient updates, thereby supporting arbitrary depth complexity without scene-specific hyperparameter tuning [S1, unofficial].

## How it works
**Inputs:** 3D Gaussian primitives (mean, covariance, opacity, spherical harmonics coefficients), camera projection matrices, image resolution.
**Outputs:** Final color buffer, depth buffer, and gradient tensors for training.

**Data Structures:**
*   **Gaussian Instances:** Each Gaussian is instantiated once per overlapping tile.
*   **Key:** A composite key combining view-space depth and tile ID, used for sorting.
*   **Tile Ranges:** For each tile, the start and end indices in the sorted global Gaussian array.
*   **Shared Memory:** Used by thread blocks to load packets of Gaussians for a tile.
*   **Per-Pixel State:** Stores the final accumulated opacity $\alpha_{final}$ and the depth of the last contributing point $z_{last}$ for backward pass optimization.

**Parameters:**
*   Tile size: $16 \times 16$ pixels [S1, unofficial].
*   Culling threshold: 99% confidence interval of the Gaussian [S1, unofficial].
*   Alpha saturation threshold: $\alpha \geq 1.0$ (or a value very close to 1, e.g., $1 - \epsilon$) [S1, unofficial].

**Steps:**

1.  **Culling:**
    *   Reject Gaussians whose 99% confidence interval does not intersect the view frustum [S1, unofficial].
    *   Apply a guard band to trivially reject Gaussians with means close to the near plane or far outside the view frustum to prevent unstable 2D covariance computation [S1, unofficial].

2.  **Instantiation and Key Assignment:**
    *   For each surviving Gaussian, determine the set of tiles it overlaps.
    *   Instantiate one Gaussian object per overlapping tile.
    *   Assign each instance a key $K = (\text{tile\_ID}, \text{view\_space\_depth})$ [S1, unofficial].

3.  **Sorting:**
    *   Perform a single global radix sort on all Gaussian instances based on their keys $K$ [S1, unofficial].
    *   This produces a globally depth-sorted list of Gaussian instances per tile.

4.  **Tile Range Identification:**
    *   Identify the start and end indices in the sorted array for each tile [S1, unofficial].

5.  **Forward Rasterization (Per Tile):**
    *   Launch one thread block per tile.
    *   Collaboratively load packets of Gaussians from global memory into shared memory [S1, unofficial].
    *   For each pixel $(u, v)$ in the tile:
        *   Initialize accumulated color $C = 0$ and accumulated opacity $\alpha = 0$.
        *   Traverse the tile's Gaussian list front-to-back.
        *   For each Gaussian, compute its 2D projection and coverage at $(u, v)$.
        *   Compute the Gaussian's alpha $\alpha_g$ and color $C_g$.
        *   Update accumulation:
            $$ \alpha_{new} = \alpha + \alpha_g (1 - \alpha) $$
            $$ C_{new} = C + C_g \alpha_g (1 - \alpha) $$
        *   Store the current $\alpha_{new}$ as the final accumulated opacity for this pixel if it is the last contributing point.
        *   Store the depth $z_g$ of the current Gaussian if it contributes to the color (i.e., $\alpha_g > 0$ and $\alpha_{new} > \alpha$).
        *   If $\alpha_{new} \geq 1.0$ (saturation), stop processing further Gaussians for this pixel [S1, unofficial].
    *   After processing all pixels, check if all pixels in the tile have saturated. If so, terminate tile processing early [S1, unofficial].

6.  **Backward Pass (Gradient Computation):**
    *   Traverse the tile's Gaussian list back-to-front [S1, unofficial].
    *   Start from the last point that affected any pixel in the tile [S1, unofficial].
    *   Collaboratively load points into shared memory [S1, unofficial].
    *   For each pixel $(u, v)$:
        *   Retrieve the final accumulated opacity $\alpha_{final}$ and the depth of the last contributing point $z_{last}$ from the forward pass.
        *   If the current Gaussian's depth $z_g > z_{last}$, skip overlap testing and processing [S1, unofficial].
        *   Otherwise, perform expensive overlap testing and compute gradients.
        *   Recover intermediate accumulated opacity $\alpha_{prev}$ for gradient computation by dividing the final accumulated opacity by the current point's alpha:
            $$ \alpha_{prev} = \frac{\alpha_{final}}{\alpha_g} $$
            This allows computation of the gradient with respect to the current Gaussian's contribution without storing the full history of opacities [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tile_based_gaussian_rasterize(gaussians, camera, tile_size=16):
    # 1. Culling
    valid_gaussians = []
    for g in gaussians:
        if not intersects_view_frustum(g, camera, threshold=0.99):
            continue
        if is_outside_guard_band(g, camera):
            continue
        valid_gaussians.append(g)
    
    # 2. Instantiation and Key Assignment
    instances = []
    for g in valid_gaussians:
        tiles = get_overlapping_tiles(g, camera, image_dims)
        for t in tiles:
            key = (t.id, g.view_space_depth)
            instances.append({
                'gaussian': g,
                'tile_id': t.id,
                'key': key,
                'start_tile_idx': t.start_idx, # Placeholder for logic
                'end_tile_idx': t.end_idx
            })
    
    # 3. Sorting
    # Sort instances by key (tile_id, then depth)
    instances.sort(key=lambda x: x['key'])
    
    # 4. Tile Ranges
    tile_ranges = {}
    for i, inst in enumerate(instances):
        tid = inst['tile_id']
        if tid not in tile_ranges:
            tile_ranges[tid] = {'start': i, 'end': i}
        else:
            tile_ranges[tid]['end'] = i
            
    # 5. Forward Rasterization
    color_buffer = zeros(image_dims)
    alpha_buffer = zeros(image_dims)
    last_contrib_depth = zeros(image_dims)
    
    for tid, range_info in tile_ranges.items():
        start_idx = range_info['start']
        end_idx = range_info['end']
        tile_gaussians = instances[start_idx:end_idx+1]
        
        # Launch thread block for tile
        # Shared memory allocation for tile_gaussians packets
        
        for y in range(tid.y * tile_size, (tid.y + 1) * tile_size):
            for x in range(tid.x * tile_size, (tid.x + 1) * tile_size):
                if y >= image_dims.height or x >= image_dims.width:
                    continue
                    
                acc_color = 0.0
                acc_alpha = 0.0
                last_z = -1.0
                
                for g_inst in tile_gaussians:
                    g = g_inst['gaussian']
                    
                    # Project and compute coverage
                    alpha_g, color_g, z_g = project_and_sample(g, x, y, camera)
                    
                    if alpha_g <= 0:
                        continue
                        
                    new_alpha = acc_alpha + alpha_g * (1.0 - acc_alpha)
                    new_color = acc_color + color_g * alpha_g * (1.0 - acc_alpha)
                    
                    acc_alpha = new_alpha
                    acc_color = new_color
                    
                    if acc_alpha > acc_alpha_prev: # Contribution detected
                        last_z = z_g
                        
                    if acc_alpha >= 1.0:
                        break
                        
                color_buffer[y, x] = acc_color
                alpha_buffer[y, x] = acc_alpha
                last_contrib_depth[y, x] = last_z
                
    return color_buffer, alpha_buffer, last_contrib_depth

def backward_pass(gaussians, instances, tile_ranges, last_contrib_depth, alpha_buffer, tile_size=16):
    # 6. Backward Pass
    for tid, range_info in tile_ranges.items():
        start_idx = range_info['start']
        end_idx = range_info['end']
        tile_gaussians = instances[start_idx:end_idx+1]
        
        # Traverse back-to-front
        for g_inst in reversed(tile_gaussians):
            g = g_inst['gaussian']
            z_g = g.view_space_depth
            
            # Check if this Gaussian could have contributed to any pixel in the tile
            # Optimization: only process if depth <= last_contrib_depth for relevant pixels
            # This requires checking per-pixel or using a tile-level bound
            
            for y in range(tid.y * tile_size, (tid.y + 1) * tile_size):
                for x in range(tid.x * tile_size, (tid.x + 1) * tile_size):
                    if y >= image_dims.height or x >= image_dims.width:
                        continue
                        
                    z_last = last_contrib_depth[y, x]
                    alpha_final = alpha_buffer[y, x]
                    
                    if z_g > z_last:
                        continue
                        
                    # Compute gradients
                    # Recover intermediate opacity
                    alpha_g, color_g, _ = project_and_sample(g, x, y, camera)
                    
                    if alpha_g > 0:
                        alpha_prev = alpha_final / alpha_g
                        # Compute gradients w.r.t. Gaussian parameters using alpha_prev
                        compute_gradients(g, alpha_prev, alpha_final, color_g)
```

## Operating characteristics
*   **Time Complexity:** Dominated by the global radix sort $O(N \log N)$ where $N$ is the number of Gaussian instances [S1, unofficial]. Rasterization is $O(T \cdot P)$ where $T$ is the number of tiles and $P$ is the number of pixels per tile, but early termination reduces effective complexity for saturated pixels [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for storing Gaussian instances and sorted arrays. Constant overhead per pixel for storing final opacity and last contributing depth [S1, unofficial].
*   **Performance:** Enables real-time rendering and efficient training on GPUs. Avoids the memory overhead of storing per-pixel lists of blended points [S1, unofficial].
*   **Strengths:** Handles arbitrary depth complexity; no hard limit on gradient updates; efficient use of GPU parallelism via tile-based processing and shared memory [S1, unofficial].
*   **Limitations:** Alpha-blending is approximate due to pre-sorting; artifacts may appear in highly complex scenes with significant depth interleaving within a tile, though these are negligible in converged scenes [S1, unofficial].
*   **When to use:** Real-time rendering and training of 3D Gaussian Splatting scenes.
*   **When not to use:** Applications requiring exact per-pixel depth sorting for alpha-blending where visual fidelity is critical over performance.

## Background
The tile-based Gaussian rasterizer was introduced in "3D Gaussian Splatting for Real-Time Radiance Field Rendering" by Kerbl et al. [S1, unofficial]. It builds on software rasterization approaches by Lassner and Zollhofer [S1, unofficial] and addresses limitations in previous alpha-blending solutions by Kopanas et al. [S1, unofficial]. The method was designed to enable efficient backpropagation over an arbitrary number of blended Gaussians with low additional memory consumption [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting for Real-Time Radiance Field Rendering [pdf], https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/3d_gaussian_splatting_low.pdf, fetched 2026-10-26 (unofficial)
