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

**Steps:**
1. **Initialization**: Load the trained NeRF checkpoint and set the model to evaluation mode (`eval()`) [S1, unofficial].
2. **Metadata Loading**: Load camera intrinsics (focal length) and extrinsics (poses, near, far) from a JSON configuration file specific to the dataset (e.g., Blender) [S1, unofficial].
3. **Trajectory Generation**: Generate a sequence of camera poses along a spiral path using `generate_spiral_path` [S1, unofficial].
4. **Ray Generation**: For each camera pose in the trajectory, compute the origin $\mathbf{o}$ and direction $\mathbf{d}$ for every pixel in the image plane using `get_rays` [S1, unofficial].
5. **Rendering Loop**:
   - Iterate through the generated poses [S1, unofficial].
   - For each pose, invoke `render_rays` with the corresponding rays [S1, unofficial].
   - Inside `render_rays`:
     - Sample $N_c$ points along each ray for coarse density estimation [S1, unofficial].
     - Sample $N_f$ points for fine color and density estimation [S1, unofficial].
     - Compute volume rendering integral and composite color [S1, unofficial].
   - Render rays in chunks to manage memory [S1, unofficial].
6. **Post-processing**:
   - Concatenate RGB chunks [S1, unofficial].
   - Clamp values to $[0, 1]$ [S1, unofficial].
   - Convert to `uint8` [S1, unofficial].
   - Append to the `frames` list [S1, unofficial].
7. **Video Export**: Write the `frames` list to an MP4 file using `imageio.mimwrite` [S1, unofficial].

**Parameters:**
- $N_c$: Number of coarse samples per ray. Default: 64 [S1, unofficial].
- $N_f$: Number of fine samples per ray. Default: 128 [S1, unofficial].
- `white_bg`: Boolean flag for background color. Default: `True` (white) for Blender dataset [S1, unofficial].

**Edge Cases and Pitfalls:**
- Precision: Floating-point accumulation during transmittance calculation can cause artifacts; standard implementations use cumulative products with careful dtype handling [G].
- Boundaries: Rays near the near/far clipping planes require explicit delta padding to avoid zero-length intervals [G].
- Degenerate input: Invalid poses or missing metadata will cause ray generation to fail; robust implementations validate JSON keys before execution [G].

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
    model.eval()
    focal = metadata['focal']
    near = metadata['near']
    far = metadata['far']
    height = metadata['height']
    width = metadata['width']
    
    poses = generate_spiral_path(trajectory_params, num_frames=100)
    frames = []
    
    for pose in poses:
        rays = get_rays(pose, height, width, focal)
        chunk_size = 1000
        rgb_chunks = []
        
        for i in range(0, rays.shape[0], chunk_size):
            chunk_rays = rays[i:i+chunk_size]
            chunk_rgb = render_rays(
                model=model, 
                rays=chunk_rays, 
                N_c=64, 
                N_f=128, 
                white_bg=True,
                near=near, 
                far=far
            )
            rgb_chunks.append(chunk_rgb)
            
        rgb = torch.cat(rgb_chunks, dim=0).reshape(height, width, 3)
        rgb = torch.clamp(rgb, 0.0, 1.0)
        frame = (rgb.cpu().numpy() * 255).astype(np.uint8)
        frames.append(frame)
        
    imageio.mimwrite('output.mp4', frames, fps=30)

def get_rays(pose, height, width, focal):
    """
    Generates camera rays from a 4x4 pose matrix.
    Returns rays of shape (height, width, 6).
    """
    i, j = torch.meshgrid(torch.arange(width), torch.arange(height), indexing='xy')
    dirs = torch.stack([(i - width / 2) / focal, -(j - height / 2) / focal, -torch.ones_like(i)], dim=-1)
    dirs = dirs / torch.norm(dirs, dim=-1, keepdim=True)
    rays_d = (dirs @ pose[:3, :3].T).reshape(height, width, 3)
    rays_o = pose[:3, 3].reshape(1, 1, 3).expand(height, width, 3)
    return torch.cat([rays_o, rays_d], dim=-1)

def generate_spiral_path(params, num_frames):
    """
    Generates a standard spiral camera trajectory for Blender dataset evaluation.
    Returns a list of 4x4 pose matrices.
    """
    poses = []
    for i in range(num_frames):
        t = i / num_frames
        angle = 2 * np.pi * t
        radius = params.get('radius', 1.0)
        height_offset = params.get('height', 0.0)
        rot = torch.tensor([
            [np.cos(angle), 0, np.sin(angle), 0],
            [0, 1, 0, height_offset],
            [-np.sin(angle), 0, np.cos(angle), 0],
            [0, 0, 0, 1]
        ])
        poses.append(rot)
    return poses

def render_rays(model, rays, N_c, N_f, white_bg, near, far):
    """
    Renders a batch of rays using coarse and fine sampling.
    rays: (N, 6)
    """
    N = rays.shape[0]
    rays_o = rays[:, :3]
    rays_d = rays[:, 3:]
    
    # Sample coarse points
    t_vals_coarse = torch.linspace(0., 1., N_c, device=rays_o.device)
    z_vals_coarse = near * (1. - t_vals_coarse) + far * t_vals_coarse
    pts_coarse = rays_o.unsqueeze(1) + rays_d.unsqueeze(1) * z_vals_coarse.unsqueeze(2)
    sigma_coarse, rgb_coarse = model(pts_coarse, rays_d)
    
    # Sample fine points
    t_vals_fine = torch.linspace(0., 1., N_f, device=rays_o.device)
    z_vals_fine = near * (1. - t_vals_fine) + far * t_vals_fine
    pts_fine = rays_o.unsqueeze(1) + rays_d.unsqueeze(1) * z_vals_fine.unsqueeze(2)
    sigma_fine, rgb_fine = model(pts_fine, rays_d)
    
    # Volume rendering
    deltas = z_vals_fine[:, 1:] - z_vals_fine[:, :-1]
    delta_inf = torch.tensor(1e10, device=rays_o.device)
    deltas = torch.cat([deltas, delta_inf.unsqueeze(0)], dim=1)
    alpha = 1.0 - torch.exp(-sigma_fine[:, :-1] * deltas)
    T = torch.cumprod(torch.exp(-sigma_fine[:, :-1] * deltas), dim=1)
    T = torch.cat([torch.ones_like(T[:, :1]), T], dim=1)
    weights = alpha * T
    rgb_final = torch.sum(weights.unsqueeze(-1) * rgb_fine[:, :-1], dim=1)
    
    if white_bg:
        bg_color = torch.tensor([1.0, 1.0, 1.0], device=rays_o.device)
        rgb_final = rgb_final + bg_color * (1.0 - torch.sum(weights, dim=1, keepdim=True))
        
    return rgb_final.reshape(-1, 3)
```

## Subtopics

### iNVS
A variant that does not use volumetric rendering. Instead, it warps a source view to the novel view and uses a Diffusion model to complete/inpaint the novel view image [S2, unofficial]. This approach is noted as being more robust to lighting inconsistencies and sparse input views compared to standard NeRF [S2, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Performance**: Rendering speed is limited by the MLP inference time. On modern GPUs, this can be slow for high-resolution videos without acceleration structures (like hash grids or octrees) [S1, unofficial].
- **Strengths**: Produces high-quality, continuous novel views; leverages the robustness of trained NeRFs for geometry and appearance [S1, unofficial].
- **Limitations**: Computationally expensive for real-time video generation; requires a pre-trained NeRF model; sensitive to input data quality (e.g., lighting consistency) unless augmented (e.g., NeRF-W) [S2, unofficial].

## Background
The method originates from the implementation of Neural Radiance Fields (NeRF) by Mildenhall et al. (ECCV 2020) [S1, unofficial]. The specific "Video Rendering Pipeline" or `render_video` script is a common utility in open-source NeRF implementations (e.g., `nerf-from-scratch`) to visualize the learned scene by traversing a spiral camera path [S1, unofficial]. The spiral path is a standard trajectory for evaluating novel view synthesis on synthetic datasets like Blender [S1, unofficial].

## Sources
[S1] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
