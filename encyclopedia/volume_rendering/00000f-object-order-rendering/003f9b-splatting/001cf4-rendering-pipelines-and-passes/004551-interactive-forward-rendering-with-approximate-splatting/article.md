# Interactive Forward Rendering with Approximate Splatting

Also called: Fast Rendering, OpenGL Approximation

## Overview
This method provides an efficient, hardware-accelerated approximation of point reprojection for high-quality free-viewpoint rendering, enabling interactive frame rates by replacing full soft rasterization with a global depth layering system [S1]. It differs from sibling methods like [Slice-by-Slice (Sheet Buffer) Splatting](2110-slice-by-slice-sheet-buffer-splatting.md) by using a fixed number of global depth layers rather than processing slices sequentially, and from [Screen Quad Rendering](2435-screen-quad-rendering.md) by explicitly managing opacity accumulation via logarithmic blending in screen space [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of input views (images and camera poses), a multi-view stereo (MVS) mesh, and a pre-trained neural network for image synthesis.
*   **Outputs:** A synthesized novel view image.

**Data Structures:**
*   **Depth Layers:** A stack of $k_d$ texture buffers representing global depth intervals. Each layer stores accumulated opacity and depth information.
*   **MVS Mesh Depth Map:** A low-resolution depth map used to determine the relevant depth interval for the scene.

**Parameters:**
*   $k_d$: Number of global depth layers. Typical value: 10 [S1].
*   Resolution: Target output resolution (e.g., 900x600) [S1].

**Steps:**
1.  **Camera Selection:** Choose a subset of input views relevant to the novel view [S1].
2.  **Depth Interval Determination:** Rasterize the MVS mesh depth at a low resolution (1/16 of target) and construct a min-max mipmap to determine the relevant depth range [S1].
3.  **Splatting into Depth Layers:** Project points from all selected input views into the $k_d$ global depth layers in parallel.
    *   Opacity accumulation uses hardware-accelerated additive blending of $\log(1 - \alpha)$ to achieve multiplicative opacity accumulation [S1].
    *   Formula for opacity accumulation: If $\alpha$ is the splat opacity, the stored value $V$ is updated as $V_{new} = V_{old} + \log(1 - \alpha)$. The final accumulated opacity is $1 - \exp(V_{total})$.
4.  **Compositing:** Composite the depth layers back-to-front in parallel over all re-projected pixels of all input images [S1].
5.  **Probabilistic Depth Test:** Apply a probabilistic depth test considering at most $k_d$ alpha-weighted depth samples to refine visibility [S1].
6.  **Neural Synthesis:** Run the neural network to synthesize the final image using the accumulated depth and color information [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def interactive_forward_rendering(input_views, mvs_mesh, neural_net, k_d=10, target_res=(900, 600)):
    # 1. Camera Selection
    selected_views = select_relevant_views(input_views)
    
    # 2. Determine Depth Interval
    low_res_depth = rasterize_mvs_mesh(mvs_mesh, resolution=(target_res[0]//16, target_res[1]//16))
    depth_min, depth_max = compute_min_max_mipmap(low_res_depth)
    
    # 3. Initialize Depth Layers
    # Each layer stores log(1-alpha) accumulation
    depth_layers = initialize_texture_stack(k_d, target_res, init_value=0.0)
    
    # 4. Splat Points into Depth Layers
    for view in selected_views:
        points = extract_points_from_view(view)
        for point in points:
            # Project point to screen and determine depth layer index
            screen_pos, depth = project_point(point, view.camera)
            layer_idx = map_depth_to_layer(depth, depth_min, depth_max, k_d)
            
            # Calculate splat opacity alpha
            alpha = compute_splat_opacity(point)
            
            # Hardware-accelerated additive blending of log(1-alpha)
            log_alpha = math.log(1 - alpha)
            depth_layers[layer_idx].additive_blend(screen_pos, log_alpha)
            
    # 5. Composite Depth Layers Back-to-Front
    # Convert log(1-alpha) back to alpha for compositing if necessary, 
    # or use the accumulated log values directly for final opacity calculation.
    # The text implies compositing happens over re-projected pixels.
    accumulated_log_alpha = depth_layers.sum() # Sum across layers
    final_alpha = 1.0 - math.exp(accumulated_log_alpha)
    
    # 6. Probabilistic Depth Test
    # Consider at most k_d alpha-weighted depth samples
    refined_depth = probabilistic_depth_test(depth_layers, k_d)
    
    # 7. Neural Network Synthesis
    # Feed accumulated depth, alpha, and color information to the neural net
    final_image = neural_net(refined_depth, final_alpha, selected_views)
    
    return final_image
```

**Edge Cases and Pitfalls:**
*   **Memory Consumption:** High memory usage due to batching all input views and depth layers. For 900x600 resolution, 10 layers, and 9 views, depth layers take ~4 GB [S1].
*   **Compression:** Memory can be reduced through standard texture compression methods [S1].

## Operating characteristics
*   **Time Complexity:** The method is designed for interactive performance. An unoptimized implementation runs at 4.5 fps [S1].
*   **Breakdown:** Camera selection: 87 ms, point splatting: 36 ms, depth test: 38 ms, network evaluation: 59 ms [S1].
*   **Memory Complexity:** High. Depth layers consume ~4 GB for 900x600 resolution, 10 layers, 9 views. PyTorch reports 4.5 GB total memory usage for this configuration [S1].
*   **Strengths:** Enables interactive viewing with marginal quality degradation compared to full soft rasterization [S1].
*   **Limitations:** High memory consumption; requires pre-trained neural network [S1].
*   **When to use:** When interactive frame rates are required for free-viewpoint rendering and sufficient GPU memory is available.
*   **When not to use:** When memory is constrained or offline rendering quality is prioritized over speed.

## Background
This method was introduced by Kopanas et al. in "Point-Based Neural Rendering with Per-View Optimization" [S1]. It is part of a differentiable multi-view pipeline used for applications such as image-based rendering (IBR), multi-view harmonization, and multi-view style transfer. The implementation was done in PyTorch with custom OpenGL-based approximations for the forward pass [S1].

## Sources
[S1] Point-Based Neural Rendering with Per-View Optimization, https://www.youtube.com/watch?v=TupOewenSAk, fetched 2026-10-31
