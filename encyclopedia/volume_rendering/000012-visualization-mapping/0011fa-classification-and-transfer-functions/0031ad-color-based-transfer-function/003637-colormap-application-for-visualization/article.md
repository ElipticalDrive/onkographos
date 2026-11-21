# Colormap Application for Visualization

Also called: Colormap Visualization, Depth Colormap Application, Render Net Image

## Overview
Colormap Application for Visualization extracts specific render items (such as alpha, normal, depth, edge, curvature, or color) from a render package and applies a colormap to single-channel data to produce a visualizable `net_image` [S1, S2, unofficial]. It serves as the final mapping step in the visualization pipeline, converting raw scalar or vector fields into perceptually uniform or distinct color representations [S1, unofficial]. It differs from [Windowing](6611-windowing.md) by focusing on semantic channel extraction and non-linear color mapping rather than linear intensity scaling, and from [Lightness-Controlled Color Harmonization](10770-lightness-controlled-color-harmonization.md) by prioritizing data fidelity over aesthetic color harmony [S1, S2, unofficial].

## How it works
**Inputs:**
- `render_items`: A dictionary or object containing available render outputs (e.g., `rend_alpha`, `rend_normal`, `surf_depth`, `render`).
- `render_mode`: A string specifying the desired output type (e.g., 'alpha', 'normal', 'depth', 'edge', 'curvature', or default).
- `colormap`: A function or lookup table that maps scalar values in $[0, 1]$ to RGB colors.
- `gradient_map`: A function or lookup table used for edge and curvature visualization.

**Outputs:**
- `net_image`: A tensor or image buffer containing the final visualized result.

**Data Structures:**
- `render_items`: Contains fields corresponding to specific render passes.
  - `rend_alpha`: Single-channel tensor (shape $[H, W]$ or $[1, H, W]$).
  - `rend_normal`: Three-channel tensor (shape $[3, H, W]$ or $[H, W, 3]$), values in $[-1, 1]$.
  - `surf_depth`: Single-channel tensor (shape $[H, W]$ or $[1, H, W]$), values in $[0, \infty)$.
  - `render`: Multi-channel tensor (typically RGB, shape $[3, H, W]$ or $[H, W, 3]$), values in $[0, 1]$.

**Steps:**
1. Determine the output mode by converting `render_mode` to lowercase: `mode = render_items[render_mode].lower()`.
2. Select the base data source based on `mode`:
   - If `mode == 'alpha'`: Set `data = rend_alpha`.
   - If `mode == 'normal'`: Set `data = rend_normal`. Normalize from $[-1, 1]$ to $[0, 1]$ using:
     $$ \text{data}_{norm} = \frac{\text{data} + 1}{2} $$
   - If `mode == 'depth'`: Set `data = surf_depth`.
   - If `mode == 'edge'`: Set `data = render`. Apply `gradient_map` to `data`.
   - If `mode == 'curvature'`: Set `data = rend_normal`. Normalize to $[0, 1]$ via $(\text{data} + 1)/2$. Apply `gradient_map` to the normalized data.
   - Else (default): Set `data = render`.
3. Check if the resulting `data` is single-channel (shape dimension 0 is 1, or total channels == 1).
   - If single-channel: Apply `colormap` to `data` to generate an RGB image.
   - If multi-channel (e.g., RGB render): Return `data` as is (assuming it is already in a visualizable format).
4. Return `net_image`.

**Pitfalls:**
- Normal and curvature modes require explicit normalization from $[-1, 1]$ to $[0, 1]$ before colormap application; failure to do so results in incorrect color mapping [S1, unofficial].
- Edge and curvature modes use `gradient_map` instead of the standard `colormap` [S1, unofficial].
- Depth visualization may require external normalization parameters (e.g., min/max depth) not handled by the core function, depending on the specific variant used [S2, unofficial][S4].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_colormap_visualization(render_items, render_mode, colormap, gradient_map):
    """
    Extracts a render item and applies a colormap for visualization.
    
    Args:
        render_items: Dict or object with keys 'rend_alpha', 'rend_normal', 
                      'surf_depth', 'render'.
        render_mode: String indicating the type of visualization.
        colormap: Function mapping scalar [0,1] to RGB.
        gradient_map: Function mapping scalar [0,1] to RGB (for edges/curvature).
        
    Returns:
        net_image: Visualized RGB tensor.
    """
    mode = render_items[render_mode].lower()
    data = None
    
    if mode == 'alpha':
        data = render_items['rend_alpha']
    elif mode == 'normal':
        data = render_items['rend_normal']
        # Normalize from [-1, 1] to [0, 1]
        data = (data + 1.0) / 2.0
    elif mode == 'depth':
        data = render_items['surf_depth']
    elif mode == 'edge':
        data = render_items['render']
        data = gradient_map(data)
    elif mode == 'curvature':
        data = render_items['rend_normal']
        # Normalize from [-1, 1] to [0, 1]
        data = (data + 1.0) / 2.0
        data = gradient_map(data)
    else:
        data = render_items['render']
        
    # Check if single channel
    # Assuming data is a tensor with shape [C, H, W] or [H, W, C]
    # We check the number of channels. If C=1, apply colormap.
    if data.shape[0] == 1: # Assuming [C, H, W] layout
        net_image = colormap(data[0]) # Apply colormap to single channel
    else:
        net_image = data
        
    return net_image
```

## Subtopics

### Proposal Depth Visualization
Visualizes depth maps from each proposal network iteration for debugging and analysis [S2, unofficial]. It applies a depth colormap to the depth output from the $i$-th proposal network, using the accumulation output for normalization [S2, unofficial]. This variant is specific to iterative proposal-based rendering architectures [S2, unofficial].

### Froxel Overlay Visualization
Builds a texture overlay visualizing the froxel grid structure [S3, unofficial]. It operates in two modes: Mode 1 (Hash) uses a hash function to assign unique colors per froxel, executed via `FFroxelOverlayHashPS` as a fullscreen pass taking `SceneColor`, `SceneDepth`, `FroxelSharedUB`, and `SvPositionToInputTextureUV` as inputs [S3, unofficial]. Mode 2 (Heatmap) uses a heatmap based on light counts, executed via `FFroxelOverlayHeatmapPS`, requiring `FroxelLists.Counts`, `Offsets`, and `LightIndices` to be non-null [S3, unofficial]. Mode 0 disables the overlay [S3, unofficial]. The `SvPositionToInputTextureUV` transform converts screen positions to input texture UVs via two basis changes [S3, unofficial].

### Depth Map Visualization Scaling
Applies linear scaling to depth maps for qualitative visualization [S4]. It uses a fixed maximum depth of 82.5 meters and a minimum depth of 1 meter for normalization [S4]. This parameter set is specific to the evaluation of dynamic urban environments using neural scene graphs [S4].

### RGB-D Silhouette Visualization
Plots ground truth and rasterized RGB, Depth, and Silhouette/Diff images for comparative analysis [S5, unofficial]. It calculates the aspect ratio from the color tensor dimensions and creates a 2x3 subplot figure [S5, unofficial]. It plots Ground Truth RGB and Depth (using 'jet' colormap, vmin=0, vmax=6) [S5, unofficial]. It clamps rasterized color to $[0, 1]$ and plots Rasterized RGB (with PSNR title) and Rasterized Depth (using 'jet' colormap, vmin=0, vmax=6) [S5, unofficial]. If a diff RGB is provided, it plots Diff RGB L1; otherwise, it plots Rasterized Silhouette [S5, unofficial]. It also plots Diff Depth L1 (using 'jet' colormap, vmin=0, vmax=6) [S5, unofficial]. The figure can be saved or logged to Weights & Biases (wandb) [S5, unofficial].

## Operating characteristics
- **Time Complexity:** The core colormap application is $O(N)$ where $N$ is the number of pixels, as it involves a single pass over the image data [S1, S2, unofficial].
- **Memory Complexity:** $O(N)$ for storing the output `net_image` [S1, unofficial].
- **Performance:** Rendering a 1920x1080 image takes approximately 16.4 seconds in the context of the dynamic urban environment benchmark, though this includes broader rendering costs, not just colormap application [S4].
- **Strengths:** Provides a unified interface for visualizing diverse render outputs (alpha, normal, depth, etc.) [S1, unofficial]. Supports specialized visualizations for debugging (proposal depth, froxel overlay) [S2, S3, unofficial].
- **Limitations:** The core function assumes specific input formats and normalization ranges (e.g., normals in $[-1, 1]$) [S1, unofficial]. Depth visualization often requires external scaling parameters not defined in the base method [S2, unofficial][S4].
- **When to use:** When debugging neural rendering pipelines, visualizing intermediate render passes, or preparing data for perceptual metrics like LPIPS [S1, S5, unofficial].
- **When not to use:** When aesthetic color harmony is the primary goal (use [Lightness-Controlled Color Harmonization](10770-lightness-controlled-color-harmonization.md)) or when linear intensity windowing is sufficient [S1, S2, unofficial].

## Background
The method originates from implementations in 2D Gaussian Splatting [S1, unofficial], Generative Neural Radiance Fields (GANerf) [S2, unofficial], Unreal Engine 5 Froxel Lighting [S3, unofficial], Multi-Level Neural Scene Graphs for Dynamic Urban Environments [S4], and SplaTAM [S5, unofficial]. It is widely used in computer vision research for debugging novel view synthesis, SLAM, and neural rendering systems [S1, S2, S3, unofficial][S4][S5, unofficial].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S3] ArminTajik/ue5-froxel-lighting, https://github.com/ArminTajik/ue5-froxel-lighting, fetched 2026-10-27 (unofficial)
[S4] Multi-Level Neural Scene Graphs for Dynamic Urban Environments, https://doi.org/10.1109/cvpr52733.2024.01996, fetched 2026-10-29
[S5] spla-tam/SplaTAM, https://github.com/spla-tam/SplaTAM, fetched 2026-10-27 (unofficial)
