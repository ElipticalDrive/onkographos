# Early Termination for Gaussian Splatting

Also called: Early Termination (ET), Early term, Full early termination

## Overview
Early Termination (ET) is a rendering acceleration method for 3D Gaussian Splatting that stops the front-to-back blending process for a pixel once its accumulated opacity or transmittance reaches a predefined threshold, thereby eliminating unnecessary computation for distant or occluded Gaussians [S1, S2]. Unlike its parent method, Early Ray Termination, which relies on strict depth sorting to ensure correct alpha-blending order, ET in Gaussian Splatting can be applied to order-independent or sort-free variants by leveraging physically grounded transmittance models that decay similarly to standard alpha-blending [S2]. It sets itself apart from siblings like Conservative Early Ray Termination by potentially offering higher throughput gains at the cost of slight visual approximations, as it may terminate rays even if the strict transmittance threshold hasn't been fully met in non-physical blending kernels [S3].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians projected onto the image plane, sorted or grouped by tile/pixel; accumulated color $C$ and transmittance $T$ per pixel; a termination threshold $\epsilon$ (typically matching the value used in vanilla 3DGS) [S1, S2].
*   **Output:** Final pixel color $C_{final}$ and a termination ratio $r^{\text{ET}}$ [S2].

**Data Structures:**
*   **Per-Pixel State:** Accumulated color $C = (R, G, B)$ and accumulated transmittance $T$. Initialized to $C = (0,0,0)$ and $T = 1.0$ [S1].
*   **Gaussian List:** A list of Gaussians intersecting the pixel's tile or ray, ordered by depth (for sorted methods) or processed in arbitrary order (for sort-free methods with physical transmittance) [S2].

**Steps:**
1.  **Initialization:** For each pixel in the tile, initialize accumulated color $C = (0,0,0)$ and transmittance $T = 1.0$ [S1].
2.  **Iteration:** Iterate through the Gaussians associated with the pixel. For each Gaussian $i$:
    a.  **Contribution Calculation:** Compute the color contribution $\Delta C_i$ and opacity $\alpha_i$ based on the Gaussian's properties and the ray intersection [S1].
    b.  **Alpha Blending Update:** Update the accumulated color and transmittance:
        $$ C_{new} = C_{old} + \Delta C_i \cdot T_{old} $$
        $$ T_{new} = T_{old} \cdot (1 - \alpha_i) $$
        [S1]
    c.  **Termination Check:** Check if the accumulated transmittance $T_{new}$ (or accumulated opacity $1 - T_{new}$) meets the termination condition.
        *   In standard 3DGS-style ET: If $T_{new} < \epsilon$, terminate processing for this pixel [S1, S2].
        *   In Cell-Based ET: If cell-level transmittance $T_{cell}^n < \epsilon$, halt blending for the cell [S2].
    d.  **Update State:** If not terminated, update $C \leftarrow C_{new}$ and $T \leftarrow T_{new}$ [S1].
3.  **Finalization:** The final pixel color is $C_{final} = C$ [S1].

**Formulas:**
*   **Termination Ratio:** The efficiency of ET is measured by the ratio of skipped Gaussians:
    $$ r^{\text{ET}} = 1 - \frac{N^{\text{rendered}}}{N^{\text{val}}} $$
    where $N^{\text{rendered}}$ is the number of Gaussians contributing to the final color, and $N^{\text{val}}$ is the total number of valid Gaussians interacted along the ray [S2].

**Parameters:**
*   **Threshold ($\epsilon$):** The transmittance threshold for early termination. Sources indicate this is maintained at the same value as used in vanilla 3DGS [S2]. Specific numerical values are not given in the sources, but it is described as an empirical threshold [S2].

**Edge Cases and Pitfalls:**
*   **Order Independence:** In sort-free methods (e.g., LC-WSR), transmittance does not strictly decrease with depth, which can lead to popping artifacts if not handled. Duplex-GS resolves this by using physically grounded transmittance that decays similarly to alpha-blending, allowing ET without explicit ordering [S2].
*   **RaySplats Cap:** In RaySplats, capping the maximum number of Gaussians can trigger early termination even if the transmittance threshold is not reached. This requires gradient correction during training to stabilize [S3].
*   **Compression Impact:** In compressed models, the ratio of unused Gaussians due to ET may decrease (e.g., from ~50% in uncompressed 3DGS to 24.3% in compressed models), reducing the potential savings [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def early_termination_render(gaussians, tile_size=16, epsilon=0.05):
    """
    Renders a tile using Early Termination.
    gaussians: List of Gaussian objects projected to the tile.
    tile_size: Size of the tile (e.g., 16x16).
    epsilon: Transmittance threshold for early termination.
    """
    # Initialize per-pixel accumulators
    # Assuming a flat array for simplicity, size = tile_size * tile_size
    num_pixels = tile_size * tile_size
    acc_color = [0.0, 0.0, 0.0] * num_pixels
    acc_transmittance = [1.0] * num_pixels
    
    # Sort Gaussians by depth if required by the specific ET variant
    # For sort-free variants with physical transmittance, sorting may not be needed
    gaussians_sorted = sort_by_depth(gaussians) 
    
    for gaussian in gaussians_sorted:
        # Identify pixels covered by this Gaussian
        covered_pixels = get_covered_pixels(gaussian, tile_size)
        
        for idx in covered_pixels:
            # Skip if already terminated
            if acc_transmittance[idx] < epsilon:
                continue
                
            # Calculate contribution
            alpha = gaussian.compute_alpha(idx)
            delta_color = gaussian.compute_color(idx)
            
            # Update accumulation
            # C_new = C_old + delta_color * T_old
            # T_new = T_old * (1 - alpha)
            
            t_old = acc_transmittance[idx]
            c_old = acc_color[idx]
            
            acc_color[idx] = [
                c_old[0] + delta_color[0] * t_old,
                c_old[1] + delta_color[1] * t_old,
                c_old[2] + delta_color[2] * t_old
            ]
            acc_transmittance[idx] = t_old * (1 - alpha)
            
            # Check for termination
            if acc_transmittance[idx] < epsilon:
                # Mark as terminated to skip future Gaussians for this pixel
                # In a real implementation, this might be handled by a mask or flag
                pass

    return acc_color
```

## Subtopics

### Early Termination for Cell Based 3DGS
A rendering optimization that halts the front-to-back blending process at the cell level once the cell-level transmittance $T_{cell}^n$ decays below a predefined threshold $\epsilon$, eliminating transparency artifacts and accelerating rendering [S2]. Unlike prior Weighted Splatting/Rendering (WSR) methods, this formulation constrains the opacity parameter $\sigma_{n,k}$ to the physically plausible range of [0, 1], allowing the empirical threshold to be directly transferred and facilitating more effective pruning and regularization [S2]. The blending is halted when $T_{cell}^n < \epsilon$ [S2].

### Alpha Pruning and Early Termination
A rasterization module subdivided into alpha-pruning, early termination, and color accumulation stages to optimize rendering by reducing point processing [S1]. The alpha-pruning stage excludes unnecessary Gaussian points based on their alpha values. The early termination stage performs multiplication and subtraction operations followed by a comparison to determine if the accumulated pixel color has reached a saturation point, beyond which further processing would not significantly alter the final output [S1]. In compressed models, this results in 24.3% of Gaussian points being unused due to early termination, saving around 24% of computational resources in subsequent stages [S1].

### RaySplats
A method that accelerates ray tracing by capping the maximum number of Gaussians accumulated along each ray, allowing storage in a fixed-size local memory array [S3]. To stabilize training under this approximation, it continues accumulating additional Gaussians beyond the cap and uses their contributions to correct the gradients of the front-most Gaussians [S3]. A key pitfall is that the cap may trigger early termination even when the transmittance threshold has not yet been reached [S3].

## Operating characteristics
*   **Throughput Gain:** Early Termination increases throughput significantly. In one implementation, it raised FPS from 97.7 to 129.0 at 1080p, a 1.32× gain (75.70% to 100.00% throughput) [S1]. In another evaluation on the Mip-NeRF360 dataset, enabling ET resulted in 155 FPS compared to 212 FPS when disabled, with a PSNR of 27.66 vs 26.60 [S2].
*   **Complexity:** Reduces the number of Gaussian points processed in the volume rendering and sorting stages. In uncompressed 3DGS, approximately 50% of Gaussian points are unused due to ET, while in compressed models, this ratio is 24.3% [S1].
*   **Strengths:** Eliminates unnecessary blending work, especially for distant or occluded Gaussians. It is compatible with both sorted and sort-free rendering pipelines if physically grounded transmittance is used [S2].
*   **Limitations:** In sort-free methods, lack of strict depth order can cause popping artifacts if transmittance is not physically grounded [S2]. Capping the number of Gaussians (as in RaySplats) can lead to premature termination and requires gradient correction [S3].
*   **When to Use:** Use when real-time performance is critical and slight visual approximations are acceptable. Particularly effective in scenes with high occlusion or large depth ranges.
*   **When Not to Use:** Avoid in scenarios requiring strict order-independent transparency without physical transmittance constraints, or when the highest possible PSNR is required without any approximation [S2].

## Background
Early Termination for Gaussian Splatting was introduced as an optimization to accelerate the rendering pipeline of 3D Gaussian Splatting. It builds on the alpha-blending principles of the original 3DGS method [S1]. The method has been adapted and extended in various forms, such as Cell-Based 3DGS (Duplex-GS) to support order-independent rendering, and hardware accelerators to reduce gate count and power consumption [S1, S2]. RaySplats introduced a capped accumulation approach for ray tracing [S3].

## Sources
[S1] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
[S2] Duplex-GS: Proxy-Guided Weighted Blending for Real-Time Order-Independent Gaussian Splatting, https://doi.org/10.1109/tcsvt.2026.3666188, fetched 2026-10-30
[S3] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
