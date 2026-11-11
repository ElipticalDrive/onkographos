# Novel View Synthesis Rendering

Also called: Video Rendering Pipeline

## Overview
This method generates a continuous video of novel views by traversing a predefined camera trajectory (typically a spiral path) and rendering each frame using a trained Neural Radiance Field (NeRF) [S1, unofficial]. It differs from static novel view synthesis by outputting a temporal sequence (MP4) rather than a single image, and from diffusion-based methods like iNVS by relying on volumetric ray marching through a learned implicit representation rather than warping and inpainting [S1, S2, unofficial].

## How it works
**Inputs:**
- A trained NeRF model (MLP mapping $(\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$) [S1, unofficial].
- Camera metadata: focal length, image dimensions, near/far clipping planes, and initial pose [S1, unofficial].
- A camera trajectory definition (e.g., spiral parameters) [S1, unofficial].

**Outputs:**
- A sequence of RGB frames concatenated into an MP4 video file [S1, unofficial].

**Data Structures:**
- `all_rays`: A 2D array of shape $(N_{total}, 6)$, where each row is $[\mathbf{o}_x, \mathbf{o}_y, \mathbf{o}_z, \mathbf{d}_x, \mathbf{d}_y, \mathbf{d}_z]$ [S1, unofficial].
- `all_rgb`: Ground truth RGB values for training validation, shape $(N_{total}, 3)$ [S1, unofficial].
- `frames`: List of rendered image arrays [S1, unofficial].

**Parameters:**
- $N_c$: Number of coarse samples per ray. Default: 64 [S1, unofficial].
- $N_f$: Number of fine samples per ray. Default: 128 [S1, unofficial].
- `white_bg`: Boolean flag for background color. Default: `True` (white) for Blender dataset [S1, unofficial].

**Steps:**
1. **Initialization**: Load the trained NeRF checkpoint and set the model to evaluation mode (`eval()`) [S1, unofficial].
2. **Metadata Loading**: Load camera intrinsics (focal length) and extrinsics (poses, near, far) from a JSON configuration file specific to the dataset (e.g., Blender) [S1, unofficial].
3. **Trajectory Generation**: Generate a sequence of camera poses along a spiral path using `generate_spiral_path` [S1, unofficial].
4. **Ray Generation**: For each camera pose in the trajectory, compute the origin $\mathbf{o}$ and direction $\mathbf{d}$ for every pixel in the image plane using `get_rays` [S1, unofficial].
5. **Rendering Loop**:
   - Iterate through the generated poses.
   - For each pose, invoke `render_rays` with the corresponding rays.
   - Inside `render_rays`:
     - Sample $N_c$ points along each ray for coarse density estimation.
     - Sample $N_f$ points for fine color and density estimation.
     - Compute volume rendering integral:
       $$ \mathbf{C}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i, \quad T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) $$
       where $\delta_j$ is the distance between samples [S1, unofficial].
     - If `white_bg` is true, add $\mathbf{c}_{bg} (1 - T_{final})$ to the result [S1, unofficial].
   - Chunk the rendering process to manage GPU memory [S1, unofficial].
6. **Post-processing**:
   - Concatenate RGB chunks along the ray axis [S1, unofficial].
   - Clamp values to $[0, 1]$ [S1, unofficial].
   - Convert to `uint8` [S1, unofficial].
   - Append to the `frames` list [S1, unofficial].
7. **Video Export**: Write the `frames` list to an MP4 file using `imageio.mimwrite` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np
import imageio

def render_novel_view_video(model, metadata, trajectory_params):
    """
    Renders a novel view synthesis video from a trained NeRF.
    """
    # 1. Load model and set to eval
    model.eval()
    
    # 2. Load metadata
    focal = metadata['focal']
    near = metadata['near']
    far = metadata['far']
    height = metadata['height']
    width = metadata['width']
    
    # 3. Generate spiral camera path
    poses = generate_spiral_path(trajectory_params, num_frames=100) # Example frame count
    
    frames = []
    
    # 4. Iterate over poses
    for pose in poses:
        # 5. Generate rays for this pose
        # get_rays returns rays of shape (H, W, 6)
        rays = get_rays(pose, height, width, focal)
        
        # 6. Render rays
        # render_rays handles chunking and volume rendering
        # Returns RGB tensor of shape (H, W, 3)
        rgb = render_rays(
            model=model, 
            rays=rays, 
            N_c=64, 
            N_f=128, 
            white_bg=True,
            near=near, 
            far=far
        )
        
        # 7. Post-process
        # Clamp to [0, 1]
        rgb = torch.clamp(rgb, 0.0, 1.0)
        # Convert to uint8
        frame = (rgb.cpu().numpy() * 255).astype(np.uint8)
        frames.append(frame)
        
    # 8. Write video
    imageio.mimwrite('output.mp4', frames, fps=30)

def render_rays(model, rays, N_c, N_f, white_bg, near, far):
    """
    Renders rays using coarse and fine sampling.
    rays: (H, W, 6)
    """
    H, W, _ = rays.shape
    rays_o = rays[:, :, :3]
    rays_d = rays[:, :, 3:]
    
    # Flatten for batch processing
    rays_o_flat = rays_o.reshape(-1, 3)
    rays_d_flat = rays_d.reshape(-1, 3)
    
    # Sample coarse points
    t_vals_coarse = torch.linspace(0., 1., N_c).to(rays_o_flat.device)
    z_vals_coarse = near * (1. - t_vals_coarse) + far * t_vals_coarse
    
    # Add noise for anti-aliasing (common in NeRF implementations)
    # [G] Standard NeRF adds uniform noise to t_vals
    noise = torch.rand_like(t_vals_coarse)
    z_vals_coarse += (noise - 0.5) / N_c
    
    pts_coarse = rays_o_flat.unsqueeze(1) + rays_d_flat.unsqueeze(1) * z_vals_coarse.unsqueeze(2)
    
    # Query NeRF for density and color (coarse)
    # [G] Assuming model returns sigma and rgb
    sigma_coarse, rgb_coarse = model(pts_coarse, rays_d_flat)
    
    # Sample fine points around high-density regions
    # [G] Standard NeRF uses importance sampling based on sigma_coarse
    # For simplicity, we assume uniform fine sampling or standard importance sampling logic
    # is encapsulated in the model or helper. Here we use uniform fine sampling for completeness
    # based on the source's high-level description.
    t_vals_fine = torch.linspace(0., 1., N_f).to(rays_o_flat.device)
    z_vals_fine = near * (1. - t_vals_fine) + far * t_vals_fine
    
    pts_fine = rays_o_flat.unsqueeze(1) + rays_d_flat.unsqueeze(1) * z_vals_fine.unsqueeze(2)
    sigma_fine, rgb_fine = model(pts_fine, rays_d_flat)
    
    # Volume Rendering
    # Compute deltas
    deltas = z_vals_fine[:, 1:] - z_vals_fine[:, :-1]
    delta_inf = torch.tensor(1e10).to(rays_o_flat.device)
    deltas = torch.cat([deltas, delta_inf], dim=1)
    
    # Alpha
    alpha = 1.0 - torch.exp(-sigma_fine * deltas)
    
    # Transmittance
    T = torch.cumprod(torch.exp(-sigma_fine * deltas), dim=1)
    T = torch.cat([torch.ones_like(T[:, :1]), T], dim=1)
    
    # Weight
    weights = alpha * T
    
    # Composite color
    rgb_final = torch.sum(weights.unsqueeze(-1) * rgb_fine, dim=1)
    
    # Background
    if white_bg:
        bg_color = torch.tensor([1.0, 1.0, 1.0]).to(rays_o_flat.device)
        rgb_final = rgb_final + bg_color * (1.0 - torch.sum(weights, dim=1, keepdim=True))
        
    return rgb_final.reshape(H, W, 3)
```

## Subtopics

### iNVS
A variant that does not use volumetric rendering. Instead, it warps a source view to the novel view and uses a Diffusion model to complete/inpaint the novel view image [S2, unofficial]. This approach is noted as being more robust to lighting inconsistencies and sparse input views compared to standard NeRF [S2, unofficial].

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of frames and pixels. Each pixel requires $N_c + N_f$ MLP queries. For $N_{frames}$ frames, $H \times W$ resolution, the total queries are $N_{frames} \cdot H \cdot W \cdot (N_c + N_f)$ [S1, unofficial].
- **Memory Complexity**: Dominated by the NeRF model weights and the ray buffers. Chunking is used to limit peak memory usage during rendering [S1, unofficial].
- **Performance**: Rendering speed is limited by the MLP inference time. On modern GPUs, this can be slow for high-resolution videos without acceleration structures (like hash grids or octrees) [S1, unofficial].
- **Strengths**: Produces high-quality, continuous novel views; leverages the robustness of trained NeRFs for geometry and appearance [S1, unofficial].
- **Limitations**: Computationally expensive for real-time video generation; requires a pre-trained NeRF model; sensitive to input data quality (e.g., lighting consistency) unless augmented (e.g., NeRF-W) [S2, unofficial].

## Background
The method originates from the implementation of Neural Radiance Fields (NeRF) by Mildenhall et al. (ECCV 2020) [S1, unofficial]. The specific "Video Rendering Pipeline" or `render_video` script is a common utility in open-source NeRF implementations (e.g., `nerf-from-scratch`) to visualize the learned scene by traversing a spiral camera path [S1, unofficial]. The spiral path is a standard trajectory for evaluating novel view synthesis on synthetic datasets like Blender [S1, unofficial].

## Sources
[S1] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
