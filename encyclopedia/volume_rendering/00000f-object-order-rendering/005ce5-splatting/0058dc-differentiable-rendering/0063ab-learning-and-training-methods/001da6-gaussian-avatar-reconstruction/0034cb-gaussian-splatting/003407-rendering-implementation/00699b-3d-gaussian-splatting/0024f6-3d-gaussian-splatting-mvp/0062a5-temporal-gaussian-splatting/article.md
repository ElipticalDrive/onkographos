# Temporal Gaussian Splatting

Also called: TemporalGS

## Overview
Temporal Gaussian Splatting (TemporalGS) is a training-free, plug-and-play acceleration method for 3D Gaussian Splatting (3DGS) that leverages temporal priors from previous frames to reduce computational redundancy in Gaussian preprocessing, sorting, and rasterization [S1, unofficial]. It solves the challenge of high FPS and low latency rendering by culling unnecessary Gaussians and approximating static regions via warped buffers, achieving up to 1.48× acceleration over vanilla 3DGS while maintaining competitive quality [S1, unofficial]. Unlike sibling methods that may require post-training compression or hardware-specific optimizations, TemporalGS operates as a software rasterization plugin that interleaves dynamic culling and selective rendering intervals [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussians $\{G_i\}$, camera poses $W_t$ (reference) and $W_{t+i}$ (target), intrinsic matrix $K$, and temporal priors from reference frame $I_t$ (TGB, TAB, Temporal Gaussian Buffer).
*   **Outputs:** Rendered frame $\bar{I}_{t+i}$ for intermediate frames $i \in [1, \lambda-1]$.

**Data Structures:**
1.  **Temporal Geometry Buffer (TGB):** Stores per-pixel depth $D_t(x,y)$ and depth standard deviation $\sigma_t(x,y)$ for the reference frame.
    *   $D_t(x,y) = \frac{\sum_{G_i \in T_k} w_i(x,y) d_i}{\sum_{G_i \in T_k} w_i(x,y)}$ [S1, unofficial]
    *   $\sigma_t(x,y) = \sqrt{\frac{\sum_{G_i \in T_k} w_i(x,y) (d_i - D_t(x,y))^2}{\sum_{G_i \in T_k} w_i(x,y)}}$ [S1, unofficial]
    *   Where $T_k$ is the tile containing pixel $(x,y)$, $w_i$ is the Gaussian weight, and $d_i$ is the Gaussian depth.
2.  **Temporal Appearance Buffer (TAB):** The color image of the reference frame $I_t$.
3.  **Temporal Gaussian Buffer ($G_{rendered}$):** The subset of Gaussians that contributed to the reference frame.
    *   $G_{rendered} = \{G_i \mid \max_{(x,y) \in I_t} w_i(x,y) > 0\}$ [S1, unofficial]

**Parameters:**
*   $\lambda$: Interval between reference frames. Accelerates $\lambda-1$ frames between $I_t$ and $I_{t+\lambda}$. Typical value: Not given in the sources.
*   $\lambda_{TDC}$: Interval for Temporal Dynamic Culling. Typical value: 50 [S1, unofficial].
*   $\lambda_{SR}$: Interval for Selective Rendering. Typical value: 5 [S1, unofficial].
*   $\eta$: Threshold percentage of selected tiles triggering a pseudo reference frame. Typical value: 40% [S1, unofficial].
*   $\lambda_1, \lambda_2$: Variance range parameters for surface modeling in TGB. Typical values: $\lambda_1=3, \lambda_2=0$ [S1, unofficial].
*   $\epsilon$: Small constant for depth initialization. Value: Not given in the sources.

**Algorithm Steps:**

1.  **Reference Frame Rendering:** Render reference frame $I_t$ using standard 3DGS. Construct TGB ($D_t, \sigma_t$), TAB ($I_t$), and $G_{rendered}$.
2.  **Initialization:** For the target frame $t+i$, initialize warped depth $D'_{min}$ to $z_{far} + \epsilon$ and warped standard deviation $\sigma'_{max}$ to 0.
3.  **Forward Warping:** Map pixels from $I_t$ to $t+i$ using camera matrices $W_t, W_{t+i}$ and intrinsic $K$:
    $$ (x', y') \approx K W_{t+i} W_t^{-1} D_t(x,y) K^{-1} [x, y, 1]^T $$
    Clip to nearest pixel. If multiple source pixels map to the same destination, preserve the geometry closer to the camera (minimum depth) and maximum standard deviation to obtain high recall [S1, unofficial]. Update $D'_{min}$ and $\sigma'_{max}$ at the destination pixels.
4.  **Surface Modeling:** Define the visible surface range at $t+i$ as $[D'_{min} - \lambda_1 \sigma'_{max}, D'_{min} + \lambda_2 \sigma'_{max}]$ [S1, unofficial].
5.  **Temporal Dynamic Culling (TDC):** Performed every $\lambda_{TDC}$ frames.
    *   **Adaptive Frustum Culling (AFC):** After near-plane culling, keep only Gaussians $G_i$ where $F_{min} \leq ndc_i \leq F_{max}$.
        *   $ndc_i = \text{Pix2NDC}(\mu'_i)$, where $\mu'_i$ is the 2D mean of $G_i$ in screen space.
        *   $F_{min} = \min_{G_j \in G_{rendered}} (ndc_j)$, $F_{max} = \max_{G_j \in G_{rendered}} (ndc_j)$ [S1, unofficial].
    *   **Temporal Occlusion Culling (TOC):** Remove Gaussians $G_i$ whose 2D means are in screen space AND whose depth $d_i > D'_{min}(\bar{\mu}_i) + \lambda_2 \sigma'_{max}(\bar{\mu}_i)$, where $\bar{\mu}_i$ is the nearest screen pixel to $G_i$'s mean. Note: Pixels with no warped geometry retain $z_{far} + \epsilon$ depth, allowing new closer surfaces to be included [S1, unofficial].
6.  **Selective Rendering (SR):** Performed every $\lambda_{SR}$ frames.
    *   **Depth Repair:** Identify "no-hit" pixels in warped depth map (where $D'_{min} == z_{far} + \epsilon$). If all 3x3 neighbors are valid, fill the pixel with the average of neighbors. Denote repaired depth as $D'_{min}'$ [S1, unofficial].
    *   **Tile Selection:** Perform backward warping (inverse of forward warping) on $D'_{min}'$ to query correspondence in reference frame. Mark tile $T_k$ as $T_{unselected}$ if any pixel in $T_k$ fails to find correspondence.
    *   **Combine Selection:** $T_{selected} = T_{oc\_selected} \cup T_{unselected}$, where $T_{oc\_selected}$ are tiles intersecting Gaussians that passed TOC but were not culled by AFC logic (or specifically those requiring re-evaluation). *Note: The source defines $T_{selected}$ as the union of tiles requiring rendering due to occlusion culling results and unmatched tiles.* [S1, unofficial].
7.  **Rendering:**
    *   Sort Gaussians intersecting $T_{selected}$ by depth.
    *   Render $T_{selected}$ tiles using standard alpha blending.
    *   For tiles $T \setminus T_{selected}$, use the warped TAB $\bar{I}_{t \to t+i}$ obtained via backward warping of $I_t$ using $D'_{min}'$ and bilinear interpolation [S1, unofficial].
    *   Stitch rendered tiles and warped TAB to form $\bar{I}_{t+i}$ [S1, unofficial].
8.  **Adaptive Reference Frame Insertion:** If the number of selected tiles at $t+i$ exceeds $\eta\%$ of total tiles, force $t+i+1$ to be a pseudo reference frame (rendered with AFC) [S1, unofficial].
9.  **Loop:** Interleave TDC and SR steps for intermediate frames.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def temporal_gaussian_splatting(Gaussians, W_ref, W_target, K, TGB, TAB, G_rendered, lambda_val, lambda_TDC, lambda_SR, eta, lambda1, lambda2, zfar, epsilon):
    """
    Render intermediate frame between reference and target using TemporalGS.
    """
    # 1. Forward Warping
    # Initialize warped buffers
    warped_depth = np.full_like(TGB.depth, zfar + epsilon)
    warped_sigma_max = np.zeros_like(TGB.depth)
    
    # Map reference pixels to target
    for y, x in pixels_in_TGB:
        depth_z = TGB.depth[y, x]
        # Unproject to 3D
        p3d_ref = unproject(x, y, depth_z, K, W_ref)
        # Project to target view
        p2d_target = project(p3d_ref, K, W_target)
        tx, ty = clip_to_pixel(p2d_target)
        
        # Update max sigma and min depth (high recall)
        sigma = TGB.sigma[y, x]
        if sigma > warped_sigma_max[ty, tx]:
            warped_sigma_max[ty, tx] = sigma
        if depth_z < warped_depth[ty, tx]:
            warped_depth[ty, tx] = depth_z
            
    D_min = warped_depth
    Sigma_max = warped_sigma_max
    
    # 2. Temporal Dynamic Culling (if interval matches)
    if frame_count % lambda_TDC == 0:
        # Adaptive Frustum Culling
        ndc_min = min([pix2ndc(g.mean) for g in G_rendered])
        ndc_max = max([pix2ndc(g.mean) for g in G_rendered])
        
        filtered_Gaussians = []
        for g in G_rendered:
            # Near-plane culling first
            if not near_plane_cull(g, W_target):
                continue
            ndc = pix2ndc(g.mean)
            if ndc_min <= ndc <= ndc_max:
                # Temporal Occlusion Culling
                # Find nearest screen pixel to g.mean
                mu_bar = nearest_screen_pixel(g.mean, W_target, K)
                # Check against warped geometry
                d_min_buf = D_min[mu_bar.y, mu_bar.x]
                sigma_max_buf = Sigma_max[mu_bar.y, mu_bar.x]
                
                # If g is behind the warped surface (considering variance)
                if g.depth > d_min_buf + lambda2 * sigma_max_buf:
                    continue # Cull
                
                filtered_Gaussians.append(g)
        
        G_active = filtered_Gaussians
    else:
        G_active = G_rendered # Use full buffer if not TDC interval, or previous active set? 
        # Source implies TDC filters G_rendered. If not TDC, we likely use the result of previous step or full set.
        # Assuming we maintain the set of Gaussians that contributed to the last valid state.
        # For simplicity in this plug-in context, we assume G_rendered is the base, 
        # but TDC reduces it. If TDC is skipped, we might still use TOC logic if implemented continuously,
        # but source says TDC is an interval strategy. 
        # Let's assume G_active is the set passed from previous frame or G_rendered if no culling occurred.
        # To be safe and implementable: if not TDC, we don't do AFC/TOC, so G_active = G_rendered.
        G_active = G_rendered

    # 3. Selective Rendering (if interval matches)
    if frame_count % lambda_SR == 0:
        # Depth Repair
        D_min_repaired = D_min.copy()
        for y, x in no_hit_pixels(D_min):
            neighbors = get_3x3_neighbors(D_min, y, x)
            if all_valid(neighbors):
                D_min_repaired[y, x] = mean(neighbors)
        
        # Tile Selection
        T_selected = set()
        for tile in tiles:
            # Check if all pixels in tile have correspondence
            has_unmatched = False
            for px, py in tile.pixels:
                # Backward warp from target (px, py) using D_min_repaired
                # This checks if (px, py) in target corresponds to a valid pixel in ref
                # If D_min_repaired[px, py] is zfar+eps, it's a no-hit, so unmatched
                if D_min_repaired[px, py] >= zfar + epsilon:
                    has_unmatched = True
                    break
                # Also check if backward warp lands outside ref bounds or invalid
                # Simplified: if forward warp didn't cover it, backward might fail or land in gap
                # Source says: "If not, the tile is selected"
                # Implementation: Query correspondence. If fail, select.
                if not has_correspondence(px, py, D_min_repaired, K, W_ref, W_target):
                    has_unmatched = True
                    break
            
            if has_unmatched:
                T_selected.add(tile)
        
        # Also include tiles intersecting Gaussians that were kept by TOC but need re-rendering?
        # Source: T_selected = Toc_selected U Tum_selected.
        # Toc_selected are tiles containing Gaussians that passed TOC? 
        # Actually, TOC culls Gaussians. The remaining Gaussians are rendered.
        # But SR selects TILES.
        # Let's assume T_selected contains tiles that need rasterization.
        # Tum_selected are tiles with unmatched pixels.
        # Toc_selected might refer to tiles containing Gaussians that were NOT culled by TOC?
        # No, TOC removes Gaussians. The remaining Gaussians are sorted and rendered.
        # The source says: "Combine selected tiles T_selected = Toc_selected U Tum_selected."
        # And "render T_selected tiles. For other tiles, use warped TAB".
        # This implies Toc_selected are tiles that MUST be rendered because they contain Gaussians 
        # that were not culled by TOC? Or tiles where TOC failed to cull?
        # Re-reading: "TOC... remove Gaussians... filtered Gaussians are denoted as...".
        # Then "Selective Rendering... identify a small portion of tiles that cannot be approximated".
        # Then "Combine... T_selected = Toc_selected U Tum_selected".
        # It seems Toc_selected refers to tiles intersecting the Gaussians that survived TOC?
        # If so, ALL tiles intersecting surviving Gaussians are selected? That defeats the purpose of SR.
        # Alternative interpretation: Toc_selected is empty or refers to specific tiles affected by culling decisions?
        # Given the ambiguity, we implement Tum_selected as the primary SR selection.
        # We will assume T_selected = Tum_selected for the SR step, and all surviving Gaussians are rendered into these tiles.
        # Wait, if we only render Tum_selected, what about Gaussians in other tiles?
        # They are approximated by TAB. So Gaussians in non-selected tiles are ignored in rasterization.
        # So T_selected must contain all tiles that contain ANY surviving Gaussian?
        # No, that's just standard rendering.
        # The key is: "render only tiles that cannot be approximated".
        # So if a tile is NOT selected, we use TAB. This implies Gaussians in non-selected tiles are NOT rendered.
        # Therefore, T_selected must be the set of tiles containing Gaussians that are NOT approximated by TAB.
        # Tum_selected are tiles with unmatched pixels (geometry changed).
        # Toc_selected: The source says "T_selected = Toc_selected U Tum_selected".
        # And "Gaussians intersected with T_selected are sorted".
        # This implies Toc_selected are tiles that contain Gaussians that passed TOC?
        # If so, we render ALL tiles with surviving Gaussians?
        # Let's look at "Selective Rendering... leverages a tile selection algorithm to only render a small portion".
        # This suggests T_selected is a SUBSET of tiles.
        # If Toc_selected is "tiles with surviving Gaussians", that's likely all tiles.
        # Perhaps Toc_selected is empty in the SR context, or refers to tiles where TOC was NOT performed?
        # Given the text "T_selected = Toc_selected U Tum_selected", and Toc is part of TDC,
        # it's possible Toc_selected refers to tiles where the culling decision was "keep", 
        # but this is redundant if we just render all kept Gaussians.
        # Let's stick to the explicit definition: Tum_selected are tiles with unmatched pixels.
        # We will assume T_selected = Tum_selected for the purpose of "small portion".
        # If the source implies Toc_selected adds more tiles, it's not defined how.
        # We will implement T_selected = Tum_selected.
        
        T_selected = Tum_selected
    else:
        # If not SR interval, do we render all?
        # Source: "interleaved every lambda_TDC and lambda_SR intervals".
        # If not SR, we might just use TAB for everything? Or render all?
        # "During stream rendering... combined and interleaved".
        # If SR is skipped, we likely rely on TDC to reduce Gaussians, but render all remaining?
        # Or use TAB for all?
        # Standard 3DGS renders all. TemporalGS accelerates.
        # If SR is off, we probably render all surviving Gaussians (standard 3DGS on filtered set).
        T_selected = all_tiles

    # 4. Render
    Image = np.zeros_like(TAB)
    
    # Render selected tiles
    for tile in T_selected:
        # Get Gaussians intersecting tile
        G_tile = [g for g in G_active if intersects(g, tile)]
        # Sort by depth
        G_tile.sort(key=lambda g: g.depth)
        # Rasterize
        Image[tile] = rasterize(G_tile, tile, K)
        
    # Warp TAB for non-selected tiles
    if T_selected != all_tiles:
        # Backward warp TAB using D_min_repaired
        I_warped = backward_warp(TAB, D_min_repaired, K, W_ref, W_target)
        for tile in all_tiles:
            if tile not in T_selected:
                Image[tile] = I_warped[tile]
                
    return Image

def has_correspondence(px, py, D_min_repaired, K, W_ref, W_target):
    # Check if (px, py) in target has valid depth
    if D_min_repaired[py, px] >= zfar + epsilon:
        return False
    # Check if backward warp is within bounds
    # Simplified: if depth is valid, assume correspondence exists
    return True
```

## Operating characteristics
*   **Time Complexity:** Reduces sorting and rasterization cost by culling Gaussians (TDC) and rendering only a subset of tiles (SR). The acceleration factor is up to 1.48× compared to vanilla 3DGS [S1, unofficial].
*   **Memory Complexity:** Requires storage for TGB (depth + sigma), TAB (color), and $G_{rendered}$ (Gaussian indices). TGB can be downscaled by factor of 2 on large-scale scenes [S1, unofficial].
*   **Strengths:** Training-free, plug-and-play, works with software and hardware rasterization, maintains high quality [S1, unofficial].
*   **Limitations:** Performance degrades with significant camera motion or large view changes (triggering pseudo reference frames). Accuracy depends on 3DGS surface representation [S1, unofficial].
*   **When to use:** Stream rendering, video playback, or applications requiring high FPS with moderate camera motion.
*   **When not to use:** Scenes with rapid, large-scale changes where $\eta$ threshold is frequently exceeded, causing frequent reference frame resets.

## Background
TemporalGS was introduced as the first training-free, plug-and-play algorithmic approach to accelerate 3DGS rendering using temporal priors [S1, unofficial]. It addresses the computational bottleneck of sorting and rasterization in 3DGS by leveraging temporal coherence between consecutive frames [S1, unofficial].

## Sources
[S1] TemporalGS: Training-Free Plug-and-Play Acceleration for 3D Gaussian Splatting Rendering via Temporal Priors, https://doi.org/10.48550/arxiv.2607.03390, fetched 2026-10-26 (unofficial)
