# Batched Ray Sampling and Neural Rendering

Also called: Create Samples, Inference Rendering, NeRF inference loop, Volume Rendering Sample Generation

## Overview
A procedure to render RGB, Depth, and Accumulation maps for a set of latent codes and camera poses using a pre-trained NeRF generator [S1, unofficial]. It handles batching and post-processing of outputs, reshaping them into image tensors [S1, unofficial]. It differs from standard ray marching by operating on a batch of latent vectors and camera poses simultaneously, optimizing memory usage and throughput for inference [S1, unofficial].

## How it works
**Inputs:**
- Latent codes $z$: Tensor of shape $(N, D_z)$, where $N$ is the number of samples and $D_z$ is the latent dimension.
- Camera poses: Optional. If provided, a list or tensor of camera extrinsics. If not provided, rays are set to `None` (implying fixed or default rays) [S1, unofficial].
- Generator: A pre-trained neural network (NeRF) capable of querying color and density given a latent code and ray.
- `batch_size`: Integer controlling the chunking of latent codes and rays for memory efficiency [S1, unofficial].
- `img_size`: Spatial dimensions $(H, W)$ of the output image [S1, unofficial].

**Outputs:**
- RGB: Tensor of shape $(N, 3, H, W)$ [S1, unofficial].
- Depth: Tensor of shape $(N, 1, H, W)$ [S1, unofficial].
- Accumulation: Tensor of shape $(N, 1, H, W)$ [S1, unofficial].

**Data Structures:**
- Rays: Input rays are structured as $(B, 2, H \times W, C)$ where $B$ is the batch size, $2$ represents the ray origin and direction, $H \times W$ is the number of pixels, and $C=3$ is the dimensionality of origin/direction vectors [S1, unofficial].
- Intermediate Query Output: The generator returns tensors of shape $(N \times H \times W, C_{out})$ where $C_{out}$ includes RGB (3), Disparity (1), Accumulation (1), etc. [S1, unofficial].

**Steps:**
1. **Initialization**: Set the generator to evaluation mode (`eval()`) to disable dropout and batch normalization updates [S1, unofficial]. Determine the device (CPU/GPU) [S1, unofficial].
2. **Batch Splitting**: Split the latent codes $z$ into batches of size `batch_size` [S1, unofficial].
3. **Ray Generation**: If camera poses are provided, generate rays for each pose using a ray sampler (e.g., `val_ray_sampler`) [S1, unofficial]. Stack and split these rays into batches matching the latent code batches [S1, unofficial]. If no poses are provided, the rays list is `None` [S1, unofficial].
4. **Iteration**: Iterate through each batch of latent codes $z_i$ and rays $rays_i$ [S1, unofficial].
5. **Gradient Disabling**: Disable gradient computation using `torch.no_grad()` [S1, unofficial].
6. **Ray Preprocessing**: Permute and flatten the rays tensor from shape $(B, 2, H, W, C)$ to $(2, B \times H \times W, 3)$ [S1, unofficial]. This flattens the spatial dimensions to query the network pixel-by-pixel within the batch [S1, unofficial].
7. **Network Query**: Pass the latent code $z_i$ and preprocessed rays $rays_i$ to the generator [S1, unofficial]. Retrieve RGB, Disparity, Accumulation, and other outputs [S1, unofficial].
8. **Reshaping**: Reshape the outputs from $(N \times H \times W, C)$ to $(N, C, H, W)$ [S1, unofficial]. This is typically done using a lambda function or view/permute operations [S1, unofficial].
9. **Collection**: Append the reshaped tensors to lists for RGB, Disparity, and Accumulation [S1, unofficial].
10. **Concatenation**: Concatenate all batch results along the first dimension to form the final tensors [S1, unofficial].
11. **Depth Conversion**: Convert disparity maps to depth maps [S1, unofficial].
12. **Return**: Return the final RGB, Depth, and Accumulation tensors [S1, unofficial].

**Formulas:**
- **Disparity to Depth Conversion**:
  $$ depth = \frac{1}{\max(10^{-10}, disparity)} $$
  Undefined values (where disparity equals $10^{10}$) are set to the far plane value [S1, unofficial].
- **Depth Normalization** (for visualization):
  $$ depth_{norm} = \frac{depth - near}{far - near} $$
  Where $near$ and $far$ are the near and far plane distances retrieved from `generator.render_kwargs_test` [S1, unofficial].

**Parameters:**
- `near`: Near plane distance, retrieved from `generator.render_kwargs_test['near']` [S1, unofficial].
- `far`: Far plane distance, retrieved from `generator.render_kwargs_test['far']` [S1, unofficial].
- `batch_size`: Controls chunking. Typical values depend on GPU memory, e.g., 1 or small integers [S1, unofficial].
- `img_size`: Spatial dimensions, e.g., 64x64 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def batched_ray_sampling_and_neural_rendering(generator, latent_codes, poses, batch_size, img_size, device):
    """
    Renders RGB, Depth, and Accumulation for a batch of latent codes and poses.
    
    Args:
        generator: Pre-trained NeRF generator model.
        latent_codes: Tensor of shape (N, D_z).
        poses: Tensor of camera poses (N, 4, 4) or None.
        batch_size: Integer for batching.
        img_size: Integer, spatial dimension (H=W).
        device: torch.device.
        
    Returns:
        rgb: Tensor (N, 3, H, W)
        depth: Tensor (N, 1, H, W)
        acc: Tensor (N, 1, H, W)
    """
    generator.eval()
    N = latent_codes.shape[0]
    
    # Split latent codes into batches
    z_batches = torch.split(latent_codes, batch_size, dim=0)
    
    # Generate rays if poses are provided
    rays_batches = None
    if poses is not None:
        # Assuming generator has a val_ray_sampler attribute
        # get_rays returns rays for a single pose
        # We need to generate rays for all poses and split them
        all_rays = []
        for pose in poses:
            # get_rays(img_size, img_size, focal, pose)
            # Note: focal is assumed to be in generator.focal
            rays = generator.val_ray_sampler(img_size, img_size, generator.focal, pose)[0]
            all_rays.append(rays)
        all_rays = torch.stack(all_rays, dim=0) # Shape: (N, 2, H, W, C)
        rays_batches = torch.split(all_rays, batch_size, dim=0)
        
    rgb_list = []
    disp_list = []
    acc_list = []
    
    for i, z_i in enumerate(z_batches):
        rays_i = rays_batches[i] if rays_batches is not None else None
        bs = z_i.shape[0]
        
        with torch.no_grad():
            # Preprocess rays: Bx2x(HxW)xC -> 2x(BxHxW)x3
            if rays_i is not None:
                # Permute to (B, H, W, 2, C) then flatten spatial and view dims
                # Original shape: (B, 2, H, W, C)
                # Permute: (B, H, W, 2, C)
                rays_i = rays_i.permute(0, 2, 3, 1, 4)
                # Flatten to (B*H*W, 2, C)
                rays_i = rays_i.reshape(bs * img_size * img_size, 2, 3)
                # Permute to (2, B*H*W, 3) as per source description
                rays_i = rays_i.permute(1, 0, 2)
            
            # Query generator
            # generator(z_i, rays=rays_i) returns rgb, disp, acc, ...
            rgb_i, disp_i, acc_i, _ = generator(z_i, rays=rays_i)
            
            # Reshape outputs from (N*H*W, C) to (N, C, H, W)
            # rgb_i shape: (bs * H * W, 3)
            # We need to reshape to (bs, H, W, 3) then permute to (bs, 3, H, W)
            reshape = lambda x: x.view(bs, img_size, img_size, x.shape[1]).permute(0, 3, 1, 2)
            
            rgb_list.append(reshape(rgb_i).cpu())
            disp_list.append(reshape(disp_i).cpu())
            acc_list.append(reshape(acc_i).cpu())
            
    # Concatenate results
    rgb = torch.cat(rgb_list, dim=0)
    disp = torch.cat(disp_list, dim=0)
    acc = torch.cat(acc_list, dim=0)
    
    # Convert disparity to depth
    # near and far from generator.render_kwargs_test
    near = generator.render_kwargs_test['near']
    far = generator.render_kwargs_test['far']
    
    # Handle undefined disparity values (10^10)
    # Set to far plane
    disp[disp == 1e10] = far
    
    # Depth = 1 / max(1e-10, disp)
    depth = 1.0 / torch.clamp(disp, min=1e-10)
    
    return rgb, depth, acc
```

**Edge Cases and Pitfalls:**
- **Undefined Disparity**: Disparity values equal to $10^{10}$ indicate undefined depth (e.g., background). These must be explicitly handled by setting them to the far plane value before or after the inverse operation [S1, unofficial].
- **Precision**: Use `max(10^{-10}, disparity)` to avoid division by zero or numerical instability with very small disparity values [S1, unofficial].
- **Batch Size**: If `batch_size` is too large, GPU memory may be exhausted. The method splits latent codes and rays to mitigate this [S1, unofficial].
- **Missing Poses**: If poses are not provided, rays must be handled correctly (e.g., `None` or default rays) to avoid errors in the generator query [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of pixels and the number of latent codes, $O(N \cdot H \cdot W)$. The batching does not change the asymptotic complexity but improves constant factors due to parallelization on GPU [S1, unofficial].
- **Memory Complexity**: Linear with respect to the batch size and image resolution. Batching reduces peak memory usage compared to processing all samples at once [S1, unofficial].
- **Performance**: Measured performance depends on the generator's complexity and hardware. For example, on a GPU with <=10GB memory, an image size of 64x64 is recommended [S1, unofficial].
- **Strengths**: Efficient for generating multiple views or latent samples. Handles batching automatically.
- **Limitations**: Requires a pre-trained generator. Memory intensive for high-resolution images or large batches.
- **When to Use**: When rendering multiple images from a NeRF model, especially for evaluation metrics like FID/KID or video generation [S1, unofficial].
- **When Not to Use**: For real-time interactive rendering where latency is critical, or when only a single view is needed (overhead of batching may not be justified).

## Background
Originated in the context of Neural Radiance Fields (NeRF) and their generative variants (e.g., GAN-based NeRFs) [S1, unofficial]. The method is part of the evaluation and inference pipeline for models that map latent codes to 3D scenes [S1, unofficial]. It is used in projects like DeepScan3D for single X-ray to 3D CT reconstruction [S1, unofficial].

## Sources
[S1] sanchit1606/DeepScan3D-Single-X-ray-to-3D-CT-Reconstruction-using-Neural-Radiance-Fields-, https://github.com/sanchit1606/DeepScan3D-Single-X-ray-to-3D-CT-Reconstruction-using-Neural-Radiance-Fields-, fetched 2026-10-26 (unofficial)
