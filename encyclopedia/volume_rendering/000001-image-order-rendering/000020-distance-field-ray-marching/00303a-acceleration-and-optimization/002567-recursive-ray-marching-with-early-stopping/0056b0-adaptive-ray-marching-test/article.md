# Adaptive Ray Marching (Test)

Also called: Iterative volume rendering, Progressive ray marching

## Overview
Adaptive Ray Marching (Test) is an iterative volume rendering technique that marches rays in batches, composites results, and stops rays that have converged (transmittance below a threshold) to save computation [S1, unofficial]. It differs from standard fixed-step ray marching by dynamically reducing the number of active rays over iterations, and from [Recursive Ray Marching with Early Stopping](3546-recursive-ray-marching-with-early-stopping.md) by processing rays in explicit batches with a fixed step count per batch rather than updating a single active set with variable steps [S1, unofficial].

## How it works
**Inputs:**
- Ray origins `rays_o` and directions `rays_d` (shape: $N_{rays} \times 3$) [S1, unofficial].
- Hit bounds `hits_t` (shape: $N_{rays} \times 2$, start and end distances) [S1, unofficial].
- Neural network model evaluating density (`sigmas`) and color (`rgbs`) at sample points [S1, unofficial].
- Background color `rgb_bg` [S1, unofficial].
- Parameters: `T_threshold` (default $10^{-4}$), `MAX_SAMPLES` (default 1024), `exp_step_factor` (default 0) [S1, unofficial].

**Outputs:**
- Final RGB image `rgb` (shape: $N_{rays} \times 3$) [S1, unofficial].
- Final opacity `opacity` (shape: $N_{rays}$) [S1, unofficial].
- Final depth `depth` (shape: $N_{rays}$) [S1, unofficial].

**Data Structures:**
- `alive_indices`: 1D tensor of indices into the ray batch that have not yet converged. Initialized to `[0, 1, ..., N_rays - 1]` [S1, unofficial].
- `opacity`, `depth`, `rgb`: Accumulators initialized to zeros [S1, unofficial].
- `samples`: Integer counter for total samples processed per ray [S1, unofficial].

**Steps:**
1. Initialize `opacity`, `depth`, `rgb` to zeros. Set `alive_indices` to all ray indices. Set `samples = 0`. Set `min_samples = 1` (or `4` if `exp_step_factor != 0`) [S1, unofficial].
2. While `samples < MAX_SAMPLES`:
   a. Let $N_{alive} = \text{len}(alive\_indices)$. If $N_{alive} == 0$, break.
   b. Calculate batch size $N_{samples} = \max(\min(N_{rays} // N_{alive}, 64), min\_samples)$ [S1, unofficial].
   c. Increment `samples` by $N_{samples}$.
   d. Generate sample points for the $N_{alive}$ rays using `raymarching_test`. This returns `xyzs` (shape: $N_{alive} \times N_{samples} \times 3$), `dirs` (shape: $N_{alive} \times N_{samples} \times 3$), `deltas` (shape: $N_{alive} \times N_{samples}$), `ts` (shape: $N_{alive} \times N_{samples}$), and `N_eff_samples` [S1, unofficial].
   e. Flatten `xyzs` and `dirs` to 2D: $(N_{alive} \times N_{samples}) \times 3$ [S1, unofficial].
   f. Compute `valid_mask` where `dirs` are not zero vectors: `valid_mask = ~torch.all(dirs==0, dim=1)` [S1, unofficial]. If `valid_mask.sum() == 0`, break.
   g. Initialize `sigmas` and `rgbs` tensors of size $(N_{alive} \times N_{samples})$ and $(N_{alive} \times N_{samples} \times 3)$ with zeros. Evaluate the neural network at `xyzs[valid_mask]` and `dirs[valid_mask]` to get `sigmas` and `rgbs`, filling only the valid entries [S1, unofficial].
   h. Reshape `sigmas` and `rgbs` back to $(N_{alive} \times N_{samples})$ and $(N_{alive} \times N_{samples} \times 3)$ respectively [S1, unofficial].
   i. Call `composite_test_fw` with `sigmas`, `rgbs`, `deltas`, `ts`, `hits_t[:, 0]`, `alive_indices`, `T_threshold`, `N_eff_samples`, and the accumulators `opacity`, `depth`, `rgb` to update them in-place [S1, unofficial].
   j. Filter `alive_indices` to keep only valid indices (`alive_indices >= 0`) [S1, unofficial].
3. After the loop, add background color: `rgb += rgb_bg * (1 - opacity)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_ray_marching_test(rays_o, rays_d, hits_t, model, rgb_bg, T_threshold=1e-4, MAX_SAMPLES=1024, exp_step_factor=0):
    N_rays = rays_o.shape[0]
    device = rays_o.device
    
    # Initialize accumulators
    opacity = torch.zeros(N_rays, device=device)
    depth = torch.zeros(N_rays, device=device)
    rgb = torch.zeros(N_rays, 3, device=device)
    
    # Initialize alive indices
    alive_indices = torch.arange(N_rays, device=device)
    samples = 0
    min_samples = 4 if exp_step_factor != 0 else 1
    
    while samples < MAX_SAMPLES:
        N_alive = len(alive_indices)
        if N_alive == 0:
            break
            
        # Calculate batch size
        N_samples = max(min(N_rays // N_alive, 64), min_samples)
        samples += N_samples
        
        # Generate sample points
        xyzs, dirs, deltas, ts, N_eff_samples = raymarching_test(
            rays_o, rays_d, hits_t[:, 0], alive_indices,
            model.density_bitfield, model.cascades, model.scale,
            exp_step_factor, model.grid_size, MAX_SAMPLES, N_samples
        )
        
        # Flatten for network evaluation
        xyzs = xyzs.reshape(-1, 3)
        dirs = dirs.reshape(-1, 3)
        
        # Filter invalid directions
        valid_mask = ~torch.all(dirs == 0, dim=1)
        if valid_mask.sum() == 0:
            break
            
        # Evaluate neural network
        sigmas = torch.zeros(len(xyzs), device=device)
        rgbs = torch.zeros(len(xyzs), 3, device=device)
        
        sigmas_valid, rgbs_valid = model(xyzs[valid_mask], dirs[valid_mask])
        sigmas[valid_mask] = sigmas_valid
        rgbs[valid_mask] = rgbs_valid.float()
        
        # Reshape back
        sigmas = sigmas.reshape(N_alive, N_samples)
        rgbs = rgbs.reshape(N_alive, N_samples, 3)
        
        # Composite and update accumulators in-place
        composite_test_fw(
            sigmas, rgbs, deltas, ts, hits_t[:, 0],
            alive_indices, T_threshold, N_eff_samples,
            opacity, depth, rgb
        )
        
        # Filter valid indices
        alive_indices = alive_indices[alive_indices >= 0]
        
    # Add background
    rgb = rgb + rgb_bg * (1 - opacity)
    
    return rgb, opacity, depth

def raymarching_test(rays_o, rays_d, hits_t, alive_indices, density_bitfield, cascades, scale, exp_step_factor, grid_size, max_samples, n_samples):
    """Generates sample points for the alive rays.
    Returns xyzs (N_alive, n_samples, 3), dirs (N_alive, n_samples, 3),
    deltas (N_alive, n_samples), ts (N_alive, n_samples), N_eff_samples (scalar)."""
    pass

def composite_test_fw(sigmas, rgbs, deltas, ts, hits_t, alive_indices, T_threshold, N_eff_samples, opacity, depth, rgb):
    """Updates opacity, depth, rgb in-place based on compositing rules.
    Filters out rays where transmittance < T_threshold."""
    pass
```

**Parameters:**
- `T_threshold`: Transmittance threshold below which a ray is considered converged. Default: $10^{-4}$ [S1, unofficial].
- `MAX_SAMPLES`: Maximum total samples per ray across iterations. Default: 1024 [S1, unofficial].

**Edge Cases and Pitfalls:**
- Zero-direction rays: Filtered by `valid_mask` to prevent division by zero or invalid sampling [S1, unofficial].
- Convergence: Rays stop contributing to computation once transmittance drops below `T_threshold`, but the loop continues until `MAX_SAMPLES` is reached for remaining rays [S1, unofficial].
- Batch size calculation: `min_samples` ensures a minimum computational load per iteration, preventing excessive overhead from very small batches [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time complexity is proportional to the total number of samples processed, which is bounded by $N_{rays} \times MAX\_SAMPLES$ [S1, unofficial].
- Memory complexity is Not given in the sources.

**Measured Performance:**
- On the `Lego` scene with an RTX 2080 Ti, the method achieved 36.2 FPS and 32.96 PSNR, compared to 18.2 FPS and 31.46 PSNR for torch-ngp [S1, unofficial].

**Scaling:**
- Performance scales with scene emptiness; faster rendering occurs when most of the scene is empty, allowing more rays to converge early [S1, unofficial].

**Strengths and Limitations:**
- Strengths: Significant speedup in transparent/empty regions due to early stopping; simple batching logic [S1, unofficial].
- Limitations: Slower than instant-ngp (60 FPS on 3090) due to iterative Python loop overhead and lack of fully fused kernels; quality may fluctuate across runs [S1, unofficial].

## Background
This method is part of the `ngp_pl` project by kwea123, an unofficial PyTorch Lightning implementation of instant-NGP [S1, unofficial]. It was developed to provide a flexible, research-friendly baseline for neural volume rendering, focusing on iterative optimization and adaptive sampling strategies [S1, unofficial].

## Sources
[S1] kwea123/ngp_pl, https://github.com/kwea123/ngp_pl, fetched 2026-10-26 (unofficial)
