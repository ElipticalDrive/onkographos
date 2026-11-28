# Volumetric Gaussian Splatting (AVIS)

Also called: Avis Gaussian Splatting, Volume-supervised 3DGS

## Overview
Volumetric Gaussian Splatting (AVIS) is a method that adapts 3D Gaussian Splatting for volume-supervised training on 3D medical-style data (CT/MR) [S1, unofficial]. It optimizes a 3D Gaussian representation against a target volume inside a mask region of interest (ROI), using volumetric and mask-aware supervision on rasterized voxel grids instead of photometric losses on rendered camera views [S1, unofficial]. Unlike the original 3DGS which uses calibrated multi-view RGB images for novel-view synthesis, AVIS uses a 3D volume plus aligned ROI mask as input, samples Gaussian initialization from voxels inside the mask (rather than sparse SfM points), and optimizes toward mask density/CT intensity via losses on rasterized voxel grids [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A target 3D volume $V$ (format: `.nii`, `.nii.gz`, `.npy`, `.mhd`) and an aligned ROI mask $M$ [S1, unofficial].
- **Output:** A set of $K$ 3D Gaussians parameterized by mean $\mu_k$, covariance $\Sigma_k$ (or Cholesky factor $L_k$), and scalar attributes (intensity $a_k$, opacity $\alpha_k$) [S1, unofficial].

**Data Structures:**
- **Volume Tensor:** Order is $[D, H, W] = [Z, Y, X]$ [S1, unofficial].
- **Gaussian Coordinates:** Normalized $[x, y, z]$ positions in $[0, 1]^3$ [S1, unofficial].
- **Attributes:** Intensity and opacity can be `learned`, `sampled`, or `sampled_mean_covered` [S1, unofficial].

**Initialization:**
1. Load volume $V$ and mask $M$.
2. Normalize volume intensities to $[0, 1]$ using $(intensity - min) / (max - min)$ [S1, unofficial].
   - **Pitfall:** If $(volume\_max - volume\_min) \le 10^{-8}$, return mid-gray ($0.5$ if normalized, else $volume\_min$) to avoid division by zero [S1, unofficial].
3. Sample initial Gaussian centers from voxels inside the mask $M$. The number of points is controlled by `--init_n_points` [S1, unofficial].

**Optimization Steps:**
1. **Rasterization:** Project Gaussians onto a voxel grid. The internal working grid can be downscaled via `--volume_render_downscale_factor` [S1, unofficial].
2. **Sampling:** Sample the Gaussian field at voxel centers. This uses trilinear interpolation via `F.grid_sample` [S1, unofficial]:
   ```python
   # Pseudocode for intensity sampling
   # volume_5d shape: [1, 1, D, H, W]
   # grid shape: [1, N, 1, 3] containing normalized [x, y, z] coordinates
   volume_5d = volume.unsqueeze(0).unsqueeze(0)
   samples = F.grid_sample(
       volume_5d,
       grid,
       mode="bilinear",
       padding_mode=padding_mode, # Default 'zeros'
       align_corners=True,
   )
   intensities = samples.view(-1, 1)
   ```
   - **Footprint Pooling (Optional):** If `enable_footprint_pooling=True` and `radius_scale > 0`, for points where `max(scale) >= max(voxel_norm)`, compute offsets in sigma units using a $[-1, 0, 1]^3$ meshgrid scaled by $0.5 \times radius\_scale$ (default 2.0). Weights are Gaussian: $\exp(-0.5 \times offset^2)$. Sample volume at $pts + scale \times offsets$, clamp to $[0, 1]$, and average using weights [S1, unofficial].
3. **Loss Computation:** Compute loss between predicted voxel intensities and target volume/mask.
   - **Supervision Target (`--supervision_target`):**
     - `'mask'`: Optimize against mask density.
     - `'ct'`: Optimize against CT intensity.
     - `'joint'`: Optimize against both.
   - **Loss Types:**
     - Mask loss: `mse`, `dice`, `tversky`, `kl` (`--volume_loss_type`) [S1, unofficial].
     - CT loss: `mse`, `dice`, `tversky`, `kl` (`--ct_loss_type`) [S1, unofficial].
   - **Weights:** `--mask_loss_weight` and `--ct_loss_weight` balance the losses in joint mode [S1, unofficial].
4. **Densification/Pruning (Optional):** If `--enable_densification` is set, update topology based on `--densify_from_iter`, `--densification_interval`, `--densify_max_new_points`, and `--prune_min_opacity` [S1, unofficial].
5. **Update:** Update Gaussian parameters using gradients from the loss.

**Parameters:**
- `--supervision_target`: `'mask'`, `'ct'`, or `'joint'` [S1, unofficial].
- `--volume_path`: Path to target volume [S1, unofficial].
- `--mask_path`: Path to aligned ROI mask [S1, unofficial].
- `--init_n_points`: Initial Gaussian count [S1, unofficial].
- `--iterations`: Number of optimization steps [S1, unofficial].
- `--radius_scale`: Multiplier for standard deviations in footprint pooling (default 2.0) [S1, unofficial].
- `--padding_mode`: Padding mode for `F.grid_sample` (default 'zeros') [S1, unofficial].
- `--normalize`: Whether to normalize intensity values to $[0, 1]$ (default True) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **GPU Nondeterminism:** GPU kernels can introduce small nondeterministic differences depending on hardware and library versions, even with seeds set [S1, unofficial].
- **Near-Constant Volume:** If $(volume\_max - volume\_min) \le 10^{-8}$, return mid-gray to avoid division by zero [S1, unofficial].
- **Hounsfield Units:** Volumes are normalized to $[0, 1]$ by the loader. Hounsfield Units are not preserved unless the loading path is changed [S1, unofficial].

## Subtopics

### R2-Gaussian Training Paradigm
A volume reconstruction method trained primarily from X-ray projections, using volumetric data only as a regularization signal, optimized to match projection consistency rather than direct volumetric fidelity [S2, unofficial]. It synthesizes X-ray projections from ground-truth volume using TIGRE toolbox, provides both synthetic projections and ground-truth volume to R2-Gaussian during reconstruction, and optimizes the network to match X-ray projections [S2, unofficial]. It achieves higher 2D PSNR than volumetric methods because it is optimized for projection matching, but yields significantly lower 3D PSNR (e.g., 22.92 vs 51.54 on Aneurysm) due to lower 3D spatial fidelity [S2, unofficial]. Training time is notably faster [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Efficient medical data visualization; direct optimization in voxel space; mask-aware supervision [S1, unofficial].
- **Limitations:** Not given in the sources.
- **When to use:** When visualizing 3D medical data (CT/MR) and supervision from volumetric ground truth is available [S1, unofficial].
- **When not to use:** When novel-view synthesis from multi-view RGB images is required (use original 3DGS) [S1, unofficial].

## Background
AVIS is a research fork of 3D Gaussian Splatting [S1, unofficial]. The original 3DGS paper, "3D Gaussian Splatting for Real-Time Radiance Field Rendering," targets novel-view synthesis from calibrated multi-view images [S1, unofficial]. AVIS retargets this to masked 3D reconstruction from medical volumes [S1, unofficial]. The corresponding paper is expected to be published in CGI conference proceedings [S1, unofficial].

## Sources
[S1] ValentinKraft/vgs, https://github.com/ValentinKraft/vgs, fetched 2026-10-27 (unofficial)
[S2] ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling, https://arxiv.org/abs/2608.05564v1, fetched 2026-10-26 (unofficial)
