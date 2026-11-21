# NeRF Rendering Pipeline

Also called: NeRF Rendering Path

## Overview
The NeRF Rendering Pipeline orchestrates the generation of novel views from a trained Neural Radiance Field by iterating over a sequence of camera poses and invoking the volume rendering primitive for each. It handles data preparation, resolution scaling, and output assembly. It differs from the core [NeRF Rendering (Alpha Compositing)](5124-nerf-rendering-alpha-compositing.md) method by managing the external loop of pose traversal, I/O, and metric computation rather than the per-ray integration itself.

## How it works
**Inputs:**
- `render_poses`: Array of shape $(N, 4, 4)$ or $(N, 3, 4)$ representing camera-to-world matrices.
- `H`, `W`: Integer height and width of the target image.
- `focal`: Scalar focal length in pixels.
- `render_kwargs`: Dictionary containing the model and rendering configuration passed to the volume renderer.
- `render_factor`: Integer downsampling factor (default 0, meaning no downsampling).
- `chunk`: Integer number of rays to process in parallel (default 32768).
- `gt_imgs`: Optional array of ground truth images for PSNR calculation.
- `savedir`: Optional directory path to save rendered images.

**Outputs:**
- `rgbs`: List or array of rendered RGB images.
- `disps`: List or array of rendered depth/disparity maps.

**Data Structures:**
- `rgbs`: List of numpy arrays, each shape $(H', W', 3)$, where $H' = H/render\_factor$ and $W' = W/render\_factor$.
- `disps`: List of numpy arrays, each shape $(H', W')$.

**Steps:**
1. **Initialize Output Containers**: Create empty lists `rgbs` and `disps`.
2. **Iterate Poses**: For each camera pose `c2w` in `render_poses`:
    a. **Downsample Resolution**: If `render_factor` > 0, compute effective height $H_{eff} = \lfloor H / render\_factor \rfloor$ and width $W_{eff} = \lfloor W / render\_factor \rfloor$. Update `focal` by dividing by `render_factor`. If `render_factor` is 0, $H_{eff}=H, W_{eff}=W$.
    b. **Render**: Call the volume rendering function with arguments $(H_{eff}, W_{eff}, focal, chunk, c2w, \text{render\_kwargs})$. This returns `rgb` (shape $H_{eff} \times W_{eff} \times 3$) and `disp` (shape $H_{eff} \times W_{eff}$).
    c. **Store Results**: Append `rgb.numpy()` to `rgbs` and `disp.numpy()` to `disps`.
    d. **Compute PSNR (Optional)**: If `gt_imgs` is not None and `render_factor` == 0:
       i. Compute Mean Squared Error: $MSE = \frac{1}{N} \sum_{i=1}^{N} (rgb_i - gt\_img_i)^2$.
       ii. Compute PSNR: $PSNR = -10 \cdot \log_{10}(MSE)$.
    e. **Save Image (Optional)**: If `savedir` is not None:
       i. Convert `rgb` to 8-bit unsigned integer format (clamping values to [0, 255]).
       ii. Save as PNG file named `'{:03d}.png'` in `savedir`.
3. **Return**: Stack `rgbs` and `disps` along axis 0 and return them.

**Parameters:**
- `render_factor`: Controls rendering speed vs. resolution. Typical values: 4 or 8 for previews. Default: 0.
- `chunk`: Controls memory usage during rendering. Default: 32768. Decrease if out-of-memory errors occur.

**Edge Cases:**
- **Memory Limits**: If `chunk` is too large for the GPU, the rendering call may fail. Reduce `chunk` size [S1, unofficial].
- **Precision**: PSNR calculation requires `render_factor` == 0 to match ground truth resolution exactly [S1, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity**: $O(N \cdot H \cdot W)$, where $N$ is the number of poses. The constant factor depends on the number of samples per ray in the underlying volume renderer.
- **Memory Complexity**: $O(N \cdot H_{eff} \cdot W_{eff})$ for storing output images. The rendering process itself uses $O(chunk \cdot S)$ memory per batch, where $S$ is samples per ray.
- **Performance**: Rendering speed is linearly proportional to the number of rays. Downsampling by `render_factor` reduces ray count by $render\_factor^2$.
- **Strengths**: Simple, modular pipeline that separates pose management from the rendering primitive.
- **Limitations**: Not optimized for real-time; relies on batched ray processing.

## Background
The pipeline is derived from the original NeRF implementation by Mildenhall et al., specifically the `render.py` script in the official repository [S1, unofficial]. It standardizes the evaluation and novel view synthesis workflow for neural radiance fields.

## Sources
[S1] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
