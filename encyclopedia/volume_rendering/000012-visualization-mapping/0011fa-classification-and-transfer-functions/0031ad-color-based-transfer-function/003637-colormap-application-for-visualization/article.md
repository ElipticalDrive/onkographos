# Colormap Application for Visualization

Also called: Colormap Classification, Colormap Visualization, Depth Colormap Application, Render Net Image

## Overview
Colormap Application for Visualization extracts and formats specific render items (alpha, normal, depth, edge, curvature, or render) from a render package for visualization [S1, unofficial]. It applies colormaps to accumulation and depth outputs to highlight data variations, such as mapping density values to distinct colors to make structural differences easier to identify [S2, unofficial][S3, unofficial]. It differs from [Windowing](6611-windowing.md) by applying perceptual color mapping to extracted scalar or vector fields rather than linear intensity scaling [S1, unofficial].

## How it works
**Inputs:**
- `render_items`: A dictionary or object containing available render outputs (e.g., `rend_alpha`, `rend_normal`, `surf_depth`, `render`) [S1, unofficial].
- `render_mode`: A string specifying the desired output type (e.g., 'alpha', 'normal', 'depth', 'edge', 'curvature', or default) [S1, unofficial].
- `colormap`: A function or lookup table that maps scalar values in $[0, 1]$ to RGB colors [S1, unofficial].
- `gradient_map`: A function or lookup table used for edge and curvature visualization [S1, unofficial].

**Outputs:**
- `net_image`: A tensor or image buffer containing the final visualized result [S1, unofficial].

**Data Structures:**
- `render_items`: Contains fields corresponding to specific render passes [S1, unofficial].
  - `rend_alpha`: Single-channel tensor (shape $[H, W]$ or $[1, H, W]$) [S1, unofficial].
  - `rend_normal`: Three-channel tensor (shape $[3, H, W]$ or $[H, W, 3]$), values in $[-1, 1]$ [S1, unofficial].
  - `surf_depth`: Single-channel tensor (shape $[H, W]$ or $[1, H, W]$), values in $[0, \infty)$ [S1, unofficial].
  - `render`: Multi-channel tensor (typically RGB, shape $[3, H, W]$ or $[H, W, 3]$), values in $[0, 1]$ [S1, unofficial].

**Steps:**
1. Determine the output mode by converting `render_mode` to lowercase: `mode = render_items[render_mode].lower()` [S1, unofficial].
2. Select the base data source based on `mode`:
   - If `mode == 'alpha'`: Set `data = rend_alpha` [S1, unofficial].
   - If `mode == 'normal'`: Set `data = rend_normal`. Normalize from $[-1, 1]$ to $[0, 1]$ using:
     $$ \text{data}_{norm} = \frac{\text{data} + 1}{2} $$
     [S1, unofficial].
   - If `mode == 'depth'`: Set `data = surf_depth` [S1, unofficial].
   - If `mode == 'edge'`: Set `data = render`. Apply `gradient_map` to `data` [S1, unofficial].
   - If `mode == 'curvature'`: Set `data = rend_normal`. Normalize to $[0, 1]$ via $(\text{data} + 1)/2$. Apply `gradient_map` to the normalized data [S1, unofficial].
   - Else (default): Set `data = render` [S1, unofficial].
3. Check if the resulting `data` is single-channel (shape dimension 0 is 1, or total channels == 1) [S1, unofficial].
   - If single-channel: Apply `colormap` to `data` to generate an RGB image [S1, unofficial].
   - If multi-channel (e.g., RGB render): Return `data` as is (assuming it is already in a visualizable format) [S1, unofficial].
4. Return `net_image` [S1, unofficial].

**Pitfalls:**
- Normal and curvature modes require explicit normalization from $[-1, 1]$ to $[0, 1]$ before colormap application; failure to do so results in incorrect color mapping [S1, unofficial].
- Edge and curvature modes use `gradient_map` instead of the standard `colormap` [S1, unofficial].
- Depth visualization may require external normalization parameters depending on the specific variant used [S2, unofficial].

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
    # 1. Get output mode string
    mode = render_items[render_mode].lower()
    data = None
    
    # 2-7. Select base data source
    if mode == 'alpha':
        data = render_items['rend_alpha']
    elif mode == 'normal':
        data = render_items['rend_normal']
        # Normalize from [-1, 1] to [0, 1] (source notes this as (net_image+1)/2)
        data = (data + 1.0) / 2.0
    elif mode == 'depth':
        data = render_items['surf_depth']
    elif mode == 'edge':
        data = render_items['render']
        data = gradient_map(data)
    elif mode == 'curvature':
        data = render_items['rend_normal']
        # Normalize to [0, 1]
        data = (data + 1.0) / 2.0
        data = gradient_map(data)
    else:
        data = render_items['render']
        
    # 8. If single channel (shape[0]==1), apply colormap
    # [G] Assumes channel-first tensor layout [C, H, W]
    if data.shape[0] == 1:
        net_image = colormap(data[0])
    else:
        net_image = data
        
    # 9. Return net_image
    return net_image
```

## Subtopics

### [Depth Map Visualization Scaling](16843-depth-map-visualization-scaling.md)
Depth Map Visualization Scaling maps raw accumulated depth values to a normalized $[0, 1]$ range for qualitative visualization using a colormap. It differs from [Log-Space Transfer Function Mapping](20110-log-space-transfer-function-mapping.md) by using a linear transformation rather than a logarithmic one, making it suitable for scenes with bounded depth ranges where linear perceptual uniformity is preferred.

### [Log-Space Transfer Function Mapping](20110-log-space-transfer-function-mapping.md)
Log-Space Transfer Function Mapping applies a logarithmic transformation to the data domain before mapping values to a colormap, preserving relative positions of control points across orders of magnitude. This method is distinct from linear transfer function mapping by compressing high dynamic range data, making it suitable for visualizing fields like density where values span several powers of ten. It builds on [Colormap Application for Visualization](7714-colormap-application-for-visualization.md) by modifying the input domain scaling.

### Proposal Depth Visualization
Proposal Depth Visualization visualizes the depth maps from each proposal network iteration for debugging and analysis [S2, unofficial]. It applies a depth colormap to the depth output from the $i$-th proposal network, using the accumulation output for normalization [S2, unofficial]. This variant is specific to iterative proposal-based rendering architectures [S2, unofficial].

### Froxel Overlay Visualization
Froxel Overlay Visualization builds a texture overlay visualizing the froxel grid structure [S4, unofficial]. Mode 1 (Hash) uses a hash function to assign unique colors per froxel, executed via `FFroxelOverlayHashPS` as a fullscreen pass taking `SceneColor`, `SceneDepth`, `FroxelSharedUB`, and `SvPositionToInputTextureUV` as inputs [S4, unofficial]. Mode 2 (Heatmap) uses a heatmap based on light counts, executed via `FFroxelOverlayHeatmapPS`, requiring `FroxelLists.Counts`, `Offsets`, and `LightIndices` to be non-null [S4, unofficial]. Mode 0 disables the overlay [S4, unofficial]. The `SvPositionToInputTextureUV` transform converts screen positions to input texture UVs via two basis changes [S4, unofficial].

### RGB-D Silhouette Visualization
RGB-D Silhouette Visualization plots ground truth and rasterized RGB, Depth, and Silhouette/Diff images for comparative analysis [S5, unofficial]. It calculates the aspect ratio from the color tensor dimensions and creates a 2x3 subplot figure [S5, unofficial]. It plots Ground Truth RGB and Depth (using 'jet' colormap, vmin=0, vmax=6) [S5, unofficial]. It clamps rasterized color to $[0, 1]$ and plots Rasterized RGB (with PSNR title) and Rasterized Depth (using 'jet' colormap, vmin=0, vmax=6) [S5, unofficial]. If a diff RGB is provided, it plots Diff RGB L1; otherwise, it plots Rasterized Silhouette [S5, unofficial]. It also plots Diff Depth L1 (using 'jet' colormap, vmin=0, vmax=6) [S5, unofficial]. The figure can be saved or logged to Weights & Biases (wandb) [S5, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Provides a unified interface for visualizing diverse render outputs (alpha, normal, depth, etc.) [S1, unofficial]. Supports specialized debugging visualizations like proposal depth maps [S2, unofficial].
- **Limitations:** The core function assumes specific input formats and normalization ranges (e.g., normals in $[-1, 1]$) [S1, unofficial]. Depth visualization may require external normalization parameters depending on the variant [S2, unofficial].
- **When to use:** When debugging neural rendering pipelines, visualizing intermediate passes, or preparing data for perceptual metrics [S1, unofficial][S5, unofficial].
- **When not to use:** Not given in the sources.

## Background
The method originates from implementations in 2D Gaussian Splatting [S1, unofficial], Generative Neural Radiance Fields (GANerf) [S2, unofficial], Unreal Engine 5 Froxel Lighting [S4, unofficial], and SplaTAM [S5, unofficial]. It is widely used in computer vision research for debugging novel view synthesis and neural rendering systems [S1, unofficial][S2, unofficial][S5, unofficial]. In industrial scanning, the technique analyzes the quality of a piece based on its density, where red represents high-density parts and blue represents noise [S3, unofficial].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S3] What in the Graphics is Volumetric Rendering?!, https://www.3denginerd.com/blog/what-in-the-graphics-is-volumetric-rendering, fetched 2026-10-27 (unofficial)
[S4] ArminTajik/ue5-froxel-lighting, https://github.com/ArminTajik/ue5-froxel-lighting, fetched 2026-10-27 (unofficial)
[S5] spla-tam/SplaTAM, https://github.com/spla-tam/SplaTAM, fetched 2026-10-27 (unofficial)
