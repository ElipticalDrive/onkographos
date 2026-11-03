# NeRF Training Loop

Also called: NeRF Training, NeRF Optimization Loop, Neural Radiance Fields Training

## Overview
The NeRF training loop optimizes a coordinate-based neural field (MLP or SIREN) to reconstruct a scene by minimizing the mean squared error (MSE) between rendered RGB values and ground truth images [S2, S6]. It iterates over epochs and batches, sampling random pixels, generating ray bundles, performing a forward pass through the network, computing loss, and updating weights via backpropagation [S3, S6].

## How it works
**Inputs:**
- Training dataset: Images, camera poses ($c2w$), and intrinsics ($K$) [S1, S3].
- Model: A neural network (MLP or SIREN) mapping 3D coordinates (and optionally viewing directions) to color and density [S1, S2].
- Hyperparameters: Learning rate, batch size, number of epochs, sampling parameters (near/far bounds, number of samples per ray) [S1, S3].

**Outputs:**
- Trained model weights and optimizer state saved as checkpoints [S5].
- Rendered images/videos for evaluation [S5].

**Data Structures:**
- `rays_rgb`: Pre-computed array of shape `[(N-1)*H*W, 3, 3]` (or `[N_rays, 3, 3]` in mini-batch). Axis 0 is flattened pixel index. Axis 1 is channel type (0: ray origin, 1: ray direction, 2: observed RGB). Axis 2 is XYZ/RGB components. Data type is float32 [S4, S7].
- `camera`: Object containing camera-to-world matrix ($c2w$), intrinsics ($K$), and image dimensions [S1, S3].
- Checkpoint dictionary: Keys are `model` (state_dict), `optimizer` (state_dict), and `epoch` (int) [S6].

**Numbered Steps:**

1. **Initialization**:
   - Parse command-line arguments and initialize summary writers for logging training and validation metrics [S1].
   - Instantiate the neural network model; if using SIREN, apply specific weight initialization (Uniform distribution scaled by $\omega_0$) [S2].
   - Initialize the Adam optimizer with learning rate `lr` [S1, S6].
   - Initialize a learning rate scheduler (`LambdaLR` with decay factor $\gamma$ and step size $s$) [S5, S6].
   - Load training and validation datasets; compute near/far bounds based on dataset type (LLFF uses bounding depths or NDC with near=0.0 and far=1.0, Blender uses fixed near=2.0 and far=6.0, DeepVoxels uses mean camera distance) [S1, S3].
   - Create a DataLoader with `batch_size=1`, `shuffle=True`, `num_workers=0`, and `collate_fn=trivial_collate` [S5].
   - If `--rvs` is enabled, replace the global render function with the reparameterized volume sampling version [S1].

2. **Training Loop** (for `epoch` from `start_epoch` to `num_epochs`):
   - **Data Loading**: Extract `image`, `camera`, and `camera_idx` from the batch; move tensors to CUDA [S5].
   - **Ray Sampling**: Sample pixel coordinates $(x, y)$ from the image grid; full-batch implementations cover the entire grid while mini-batch uses a random subset [S2, S3]. Generate ray origins and directions using camera intrinsics and poses [S1, S3]. Construct ray bundle tensor containing $r_o$, $r_d$, near, far, and optionally viewing directions if `use_viewdirs` is enabled [S1].
   - **Forward Pass**: Sample points along each ray within $[near, far]$ bounds; perturb sampling times if `perturb > 0` [S1, S6]. Pass sampled 3D coordinates (and viewing directions) through the neural network to predict volume density ($\sigma$) and color ($c$) [S1, S6]. Perform differentiable volume rendering to compute predicted RGB for each ray [S1, S6].
   - **Loss Calculation**: Compute MSE loss between predicted RGB and ground truth RGB at sampled pixels [S2, S6]. If coarse network output exists and `--rvs` is not enabled, add its MSE loss to the total loss [S1]. If using volume SDF rendering, add Eikonal loss term: $L_{eikonal} = \lambda_{eikonal} \|\nabla \phi(x) - 1\|^2$ [S5]. Compute gradients and Laplacian for visualization *before* the backward pass [S2].
   - **Backpropagation**: Zero gradients (`optimizer.zero_grad()`), compute gradients (`loss.backward()`), update weights (`optimizer.step()`) [S5].
   - **Scheduler Step**: Update learning rate via $lr_{new} = lr_{old} \times \gamma^{(epoch / s)}$ [S5, S6].
   - **Checkpointing**: If `epoch % checkpoint_interval == 0` and `epoch > 0`, save model state, optimizer state, and epoch number [S5].
   - **Evaluation**: If `epoch % render_interval == 0` and `epoch > 0`, render images using test poses and save to disk [S5].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

def train_nerf_loop(model, optimizer, scheduler, train_loader, val_loader, cfg):
    start_epoch = 0
    if cfg.resume_path:
        checkpoint = torch.load(cfg.resume_path, weights_only=False)
        model.load_state_dict(checkpoint['model'])
        optimizer.load_state_dict(checkpoint['optimizer'])
        start_epoch = checkpoint['epoch'] + 1

    for epoch in range(start_epoch, cfg.training.num_epochs):
        model.train()
        total_loss = 0.0

        for batch in train_loader:
            image = batch['image'].cuda().unsqueeze(0)
            camera = batch['camera'].cuda()

            xy_grid = get_random_pixels_from_image(
                cfg.training.batch_size, cfg.data.image_size, camera
            )

            rays_o, rays_d = get_rays_from_pixels(xy_grid, cfg.data.image_size, camera)

            ray_bundle = {
                'origins': rays_o,
                'directions': rays_d,
                'near': cfg.training.near,
                'far': cfg.training.far,
                'use_viewdirs': cfg.model.use_viewdirs
            }

            out = model(ray_bundle)
            pred_rgb = out['rgb']

            gt_rgb = sample_images_at_xy(image, xy_grid)

            loss = F.mse_loss(pred_rgb, gt_rgb)

            if cfg.renderer.type == 'volume_sdf':
                eikonal_loss = get_eikonal_loss(
                    model.implicit_fn, cfg.training.batch_size,
                    cfg.training.bounds, gt_rgb.device
                )
                loss += cfg.training.eikonal_weight * eikonal_loss

            if epoch % cfg.training.steps_til_summary == 0:
                grad = gradient(pred_rgb, ray_bundle['origins'])
                laplacian = laplace(pred_rgb, ray_bundle['origins'])

            optimizer.zero_grad()
            loss.backward()
            optimizer.step()

            total_loss += loss.item()

        scheduler.step()

        if epoch % cfg.training.checkpoint_interval == 0 and epoch > 0:
            save_checkpoint(
                model.state_dict(), optimizer.state_dict(),
                epoch, cfg.training.checkpoint_path
            )

        if epoch % cfg.training.render_interval == 0 and epoch > 0:
            render_evaluation(model, val_loader, cfg)

def get_random_pixels_from_image(batch_size, image_size, camera):
    H, W = image_size
    x = torch.randint(0, W, (batch_size,))
    y = torch.randint(0, H, (batch_size,))
    return torch.stack([x, y], dim=1)

def get_rays_from_pixels(xy_grid, image_size, camera):
    H, W = image_size
    K = camera['K']
    c2w = camera['c2w']
    K_inv = torch.inverse(K)
    ones = torch.ones_like(xy_grid[..., :1])
    pixel_coords = torch.cat([xy_grid.float(), ones], dim=-1)
    dirs_cam = torch.matmul(K_inv, pixel_coords.t()).t()
    dirs_cam = F.normalize(dirs_cam, dim=-1)
    dirs_world = torch.matmul(c2w[:3, :3], dirs_cam.t()).t()
    origins_world = c2w[:3, 3].unsqueeze(0).expand(xy_grid.shape[0], -1)
    return origins_world, dirs_world

def sample_images_at_xy(image, xy_grid):
    H, W = image.shape[1:3]
    x_norm = (xy_grid[:, 0].float() / (W - 1)) * 2 - 1
    y_norm = (xy_grid[:, 1].float() / (H - 1)) * 2 - 1
    grid = torch.stack([x_norm, y_norm], dim=-1).unsqueeze(0).unsqueeze(1)
    img_4d = image.permute(0, 3, 1, 2)
    sampled = F.grid_sample(img_4d, grid, align_corners=True)
    return sampled.squeeze(-1).squeeze(0).permute(1, 0)

def gradient(output, input):
    grad = torch.autograd.grad(
        output.sum(), input, create_graph=True, retain_graph=True
    )[0]
    return grad

def laplace(output, input):
    grad = gradient(output, input)
    div = torch.autograd.grad(
        grad.sum(), input, create_graph=True, retain_graph=True
    )[0]
    return div

def get_eikonal_loss(implicit_fn, batch_size, bounds, device):
    pts = torch.rand(batch_size, 3, device=device) * (bounds[1] - bounds[0]) + bounds[0]
    pts.requires_grad_(True)
    phi = implicit_fn(pts)
    grad = torch.autograd.grad(
        phi.sum(), pts, create_graph=True, retain_graph=True
    )[0]
    return torch.mean((torch.norm(grad, dim=1) - 1.0) ** 2)

def save_checkpoint(model_state, optimizer_state, epoch, path):
    torch.save({
        'model': model_state,
        'optimizer': optimizer_state,
        'epoch': epoch
    }, path)

def render_evaluation(model, val_loader, cfg):
    model.eval()
    with torch.no_grad():
        for batch in val_loader:
            pass
```

**Parameters:**
- `lr` (float): Learning rate for Adam optimizer [S1, S6].
- `total_steps` (int): Number of optimization steps [S2].
- `lr_scheduler_gamma` (float): Gamma factor for the LambdaLR scheduler [S5, S6].
- `lr_scheduler_step_size` (int): Step size interval for the LambdaLR scheduler [S5, S6].
- `perturb` (float): Jitter amount for sampling; 0.0 for no jitter, 1.0 for jitter [S1].
- `raw_noise_std` (float): Standard deviation of noise added to regularize $\sigma_a$ output; default 0.0, 1e0 recommended [S1].
- `multires` (int): Log2 of max frequency for positional encoding of 3D location; default 10 [S1].
- `multires_views` (int): Log2 of max frequency for positional encoding of 2D direction; default 4 [S1].
- `eikonal_weight` (float): Weight for the Eikonal loss term in volume SDF rendering [S5].
- `checkpoint_interval` (int): Frequency of checkpoint saving in epochs [S5].
- `render_interval` (int): Frequency of rendering evaluation in epochs [S5].
- `num_epochs` (int): Total number of training epochs [S5].
- `render_factor` (int): Downsampling factor to speed up rendering; set 4 or 8 for fast preview, default 0 [S1].
- `dataset_type` (str): Dataset options: llff / blender / deepvoxels; default llff [S1].
- `use_viewdirs` (bool): Use full 5D input instead of 3D [S1].
- `samples_union` (bool): Flag for samples union variant [S1].
- `rvs` (bool): Use reparameterized volume sampling version of the algorithm [S1].
- `precrop_iters` (int): Number of steps to train on central crops; default 0 [S1].
- `precrop_frac` (float): Fraction of image taken for central crops; default 0.5 [S1].
- `testskip` (int): Will load 1/N images from test/val sets; default 8 [S1].
- `shape` (str): DeepVoxels shape options: armchair / cube / greek / vase; default greek [S1].
- `white_bkgd` (bool): Set to render synthetic data on a white background [S1].
- `half_res` (bool): Load Blender synthetic data at 400x400 instead of 800x800 [S1].
- `factor` (int): Downsample factor for LLFF images; default 8 [S1].
- `no_ndc` (bool): Do not use normalized device coordinates [S1].
- `lindisp` (bool): Sampling linearly in disparity rather than depth [S1].
- `spherify` (bool): Set for spherical 360 scenes [S1].
- `llffhold` (int): Will take every 1/N images as LLFF test set; default 8 [S1].
- `i_print` (int): Frequency of console printout and metric logging; default 100 [S1].
- `i_img` (int): Frequency of TensorBoard image logging; default 500 [S1].
- `i_weights` (int): Frequency of weight checkpoint saving; default 10000 [S1].
- `i_testset` (int): Frequency of testset saving; default 50000 [S1].
- `i_video` (int): Frequency of render_poses video saving; default 50000 [S1].
- `N_rand` (int): Number of random rays to sample per batch; typical value 1024 or 2048 [S4, S7].
- `use_batching` (bool): Boolean flag determining if random ray batching is enabled [S7].
- `chunk` (int): Maximum number of rays processed per forward pass to limit VRAM usage [S4].
- `steps_til_summary` (int): Frequency of logging and plotting [S2].
- `n_layers_xyz` (int): Number of layers for 3D location MLP; default 6 [S3].
- `n_hidden_neurons_xyz` (int): Hidden neurons for 3D location MLP; default 128 [S3].
- `n_hidden_neurons_dir` (int): Hidden neurons for direction MLP; default 64 [S3].
- `append_xyz` (list): List of layers to append 3D coordinates to; default [3] [S3].

**Edge Cases and Pitfalls:**
- **Gradient Computation Order**: Higher-order derivatives (gradient, Laplacian) for logging must be computed *before* `loss.backward()` because the backward pass frees the autograd graph [S2].
- **Precision**: Use float32 for standard training; float64 may be needed for precise gradient checks but increases memory usage [G].
- **Boundary Conditions**: Ensure ray sampling respects near/far bounds to avoid NaNs in density computation [S1].
- **Degenerate Input**: Handle cases where camera intrinsics are missing by initializing with default values based on image dimensions [S1].

## Subtopics

### Training Loop Setup
Loads image data and transform matrices from paths, moving them to the specified device. Generates the initial grid using `generate_grid` with `gridsize`, `points_distance`, `info_size=4`, and device [S8].

## Operating characteristics
- Training requires multiple epochs and iterations [S3].
- Chunking controls maximum memory usage without affecting final results [S1].
- Scaling: Memory and compute scale linearly with batch size and ray samples [G].
- When to use: Not given in the sources.
- When not to use: Not given in the sources.

## Background
The training procedure involves iterative optimization of a neural radiance field by sampling random rays per iteration, rendering predicted colors via volume integration, computing MSE loss against ground truth, and updating network weights via gradient descent [S4]. It builds on coordinate-based neural representations [S2].

## Sources
[S1] GreatDrake/reparameterized-volume-sampling, https://github.com/GreatDrake/reparameterized-volume-sampling, fetched 2026-10-26, [S1, unofficial]
[S2] Yonghao-Lee/differentiable-rendering, https://github.com/Yonghao-Lee/differentiable-rendering, fetched 2026-10-26, [S2, unofficial]
[S3] omkarchittar/Neural_Radiance_Fields, https://github.com/omkarchittar/Neural_Radiance_Fields, fetched 2026-10-26, [S3, unofficial]
[S4] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26, [S4, unofficial]
[S5] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S6] Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26, [S6, unofficial]
[S7] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27, [S7, unofficial]
[S8] DanJbk/Plenoxels, https://github.com/DanJbk/Plenoxels, fetched 2026-10-27, [S8, unofficial]
