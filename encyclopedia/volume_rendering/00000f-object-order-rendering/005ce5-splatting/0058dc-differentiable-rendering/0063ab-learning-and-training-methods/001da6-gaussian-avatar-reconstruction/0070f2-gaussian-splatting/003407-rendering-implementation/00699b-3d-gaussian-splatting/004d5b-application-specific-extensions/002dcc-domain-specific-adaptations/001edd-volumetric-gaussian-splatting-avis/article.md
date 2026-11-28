# Volumetric Gaussian Splatting (AVIS)

Also called: Avis Gaussian Splatting, Volume-supervised 3DGS

## Overview
Volumetric Gaussian Splatting (AVIS) is a method for reconstructing 3D medical-style volumes (CT/MR) by optimizing a 3D Gaussian representation against a target volume and an aligned region-of-interest (ROI) mask [S1, unofficial]. Unlike standard 3D Gaussian Splatting which synthesizes novel views from multi-view RGB images, AVIS rasterizes Gaussians into a voxel grid and optimizes them using volumetric losses (e.g., MSE, Dice) on the mask density and CT intensity [S1, unofficial]. It replaces photometric supervision with direct volume supervision, initializing Gaussians from voxels within the mask rather than sparse Structure-from-Motion (SfM) points [S1, unofficial].

## How it works
**Inputs:**
- `volume_path`: Target 3D volume (formats: `.nii`, `.nii.gz`, `.npy`, `.mhd`) [S1, unofficial].
- `mask_path`: Aligned ROI mask for initialization and masked losses [S1, unofficial].
- `model_path`: Directory for checkpoints and exports [S1, unofficial].
- `supervision_target`: Optimization target: `'mask'`, `'ct'`, or `'joint'` [S1, unofficial].

**Outputs:**
- Optimized 3D Gaussian parameters (position, covariance, opacity, intensity/SH coefficients) stored in a PLY file [S1, unofficial].
- Evaluation metrics (masked MSE, PSNR) [S1, unofficial].

**Data Structures:**
- **Volume Tensor:** Shape `[D, H, W]` corresponding to `[Z, Y, X]` [S1, unofficial]. Values are normalized to `[0, 1]` by the loader [S1, unofficial]. Hounsfield Units are not preserved unless the loading path is modified [S1, unofficial].
- **Gaussian Coordinates:** Normalized positions `[x, y, z]` in `[0, 1]^3` [S1, unofficial].
- **Gaussian Attributes:** Inherits standard 3DGS attributes (position, scale, rotation, opacity) plus intensity/feature vectors [S1, unofficial].

**Parameters:**
- `init_n_points`: Initial number of Gaussians sampled from the mask (e.g., 250,000) [S1, unofficial].
- `radius_scale`: Multiplier for standard deviations in footprint pooling (default 2.0) [S1, unofficial].
- `padding_mode`: Padding mode for `F.grid_sample` (default `'zeros'`) [S1, unofficial].
- `normalize`: Whether to normalize intensity to `[0, 1]` (default `True`) [S1, unofficial].
- `supervision_target`: `'mask'`, `'ct'`, or `'joint'` [S1, unofficial].
- `volume_loss_type`: Loss for mask branch (`'mse'`, `'dice'`, `'tversky'`, `'kl'`) [S1, unofficial].
- `ct_loss_type`: Loss for CT branch (`'mse'`, `'dice'`, `'tversky'`, `'kl'`) [S1, unofficial].
- `mask_loss_weight`, `ct_loss_weight`: Weights for joint supervision [S1, unofficial].
- `enable_footprint_pooling`: Opt-in flag for footprint pooling [S1, unofficial].
- `medical_mode`: Preset behavior (`'none'`, `'organ'`, `'vessel'`, `'vessel_anisotropy'`) [S1, unofficial].
- `intensity_mode`: How per-Gaussian intensity is handled (`'learned'`, `'sampled'`, `'sampled_mean_covered'`) [S1, unofficial].

**Steps:**
1. **Initialization:** Load volume and mask. Normalize volume to `[0, 1]`. Sample initial Gaussian centers from voxels inside the mask [S1, unofficial].
2. **Rasterization:** Rasterize Gaussians into a voxel grid. If `enable_footprint_pooling` is `True` and `scale` is provided with `radius_scale > 0`:
   - For points where `max(scale) >= max(voxel_norm)`, compute offsets in sigma units using a `[-1, 0, 1]^3` meshgrid scaled by `0.5 * radius_scale` [S1, unofficial].
   - Compute weights: $w = \exp(-0.5 \cdot \text{offset}^2)$ [S1, unofficial].
   - Sample volume at `pts + scale * offsets`, clamp to `[0, 1]`, and average using weights [S1, unofficial].
3. **Intensity Sampling:** Sample volume intensities at Gaussian centers using trilinear interpolation via `F.grid_sample` [S1, unofficial]:
   ```python
   volume_5d = volume.unsqueeze(0).unsqueeze(0)
   samples = F.grid_sample(
       volume_5d,
       grid,
       mode="bilinear",
       padding_mode=padding_mode,
       align_corners=True,
   )
   intensities = samples.view(-1, 1)
   ```
   [S1, unofficial]
4. **Loss Computation:** Compute losses based on `supervision_target`:
   - If `'mask'`: Compute `volume_loss_type` between rasterized mask and target mask [S1, unofficial].
   - If `'ct'`: Compute `ct_loss_type` between rasterized intensity and target volume [S1, unofficial].
   - If `'joint'`: Combine losses using `mask_loss_weight` and `ct_loss_weight` [S1, unofficial].
5. **Optimization:** Update Gaussian parameters using gradients from the loss. Support densification/pruning if `enable_densification` is `True` [S1, unofficial].
6. **Evaluation:** Periodically compute masked MSE/PSNR using `evaluate_ply_masked_mse.py` [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Near-constant volume:** If `volume_max - volume_min <= 1e-8`, return mid-gray (0.5 if `normalize`, else `volume_min`) to avoid division by zero or meaningless gradients [S1, unofficial].
- **Nondeterminism:** GPU kernels can introduce small nondeterministic differences depending on hardware and library versions, even with seeds set [S1, unofficial].
- **Hounsfield Units:** Volumes are normalized to `[0, 1]` by default; HU values are not preserved unless the loading path is changed [S1, unofficial].
- **Coordinate System:** Volume tensor order is `[D, H, W] = [Z, Y, X]`, while Gaussian coordinates are normalized `[x, y, z]` in `[0, 1]^3` [S1, unofficial].

## Subtopics

### R2-Gaussian Training Paradigm
A volume reconstruction method trained primarily from X-ray projections, using volumetric data only as a regularization signal, optimized to match projection consistency rather than direct volumetric fidelity [S2, unofficial]. It synthesizes X-ray projections from ground-truth volume using the TIGRE toolbox and provides both synthetic projections and ground-truth volume during reconstruction [S2, unofficial]. It achieves higher 2D PSNR than volumetric methods because it is optimized for projection matching, but yields significantly lower 3D PSNR (e.g., 22.92 vs. 51.54 on Aneurysm) due to lower 3D spatial fidelity [S2, unofficial]. Training time is notably faster [S2, unofficial]. Limited to datasets up to Neuron size due to lack of support for large raw volumetric data [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Training time is notably faster than volumetric methods when using projection-based supervision (R2-Gaussian), but AVIS itself requires iterative optimization of Gaussians against a voxel grid [S2, unofficial].
- **Memory Complexity:** VRAM usage is controlled by `max_points_per_iter` and `volume_downscale_factor` [S1, unofficial]. Large volumes may require downsampling [S1, unofficial].
- **Performance:** AVIS achieves high 3D fidelity (e.g., 51.54 PSNR on Aneurysm) compared to projection-based methods [S2, unofficial].
- **Strengths:** Direct volume supervision, efficient medical data visualization, support for various loss functions (MSE, Dice, Tversky, KL) [S1, unofficial].
- **Limitations:** Requires spatially aligned volume and mask; Hounsfield Units not preserved by default; nondeterministic GPU results [S1, unofficial].
- **When to use:** Medical volume reconstruction and visualization where direct volume supervision is preferred over multi-view synthesis [S1, unofficial].
- **When not to use:** When Hounsfield Unit preservation is critical without custom loading; when large raw volumetric data exceeds memory limits without downsampling [S1, unofficial].

## Background
AVIS is a research fork of 3D Gaussian Splatting adapted for volume-supervised training on 3D medical-style data [S1, unofficial]. The original 3D Gaussian Splatting paper focused on novel-view synthesis from calibrated multi-view images [S1, unofficial]. AVIS retargets the Gaussian parameterization and optimization machinery to masked 3D reconstruction from medical volumes [S1, unofficial]. The paper is expected to be published in CGI conference proceedings [S1, unofficial].

## Sources
[S1] ValentinKraft/vgs, https://github.com/ValentinKraft/vgs, fetched 2026-10-27 (unofficial)
[S2] ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling, https://arxiv.org/abs/2608.05564v1, fetched 2026-10-26 (unofficial)
