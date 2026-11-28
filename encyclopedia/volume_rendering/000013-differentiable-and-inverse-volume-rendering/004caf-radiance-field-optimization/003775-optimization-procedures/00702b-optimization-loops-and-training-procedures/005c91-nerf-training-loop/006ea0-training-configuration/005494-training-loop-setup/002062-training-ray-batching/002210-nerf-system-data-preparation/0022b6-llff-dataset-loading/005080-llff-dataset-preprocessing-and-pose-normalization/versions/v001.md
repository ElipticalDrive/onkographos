# LLFF Dataset Preprocessing and Pose Normalization

Also called: LLFF Data Loading, LLFF Pose Recentering

## Overview
This method prepares raw LLFF (Look at Light From) dataset scenes for differentiable volume rendering by loading images, parsing COLMAP-derived poses, correcting axis conventions, rescaling geometry, recentering the camera trajectory, and converting rays to Normalized Device Coordinates (NDC). It differs from generic NeRF data loaders by specifically handling the LLFF dataset's unique metadata structure (including temperature, exposure, and COC) and its forward-facing scene constraints, which enable the use of NDC normalization [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- Raw image files from an LLFF scene directory.
- `poses_bounds.npy`: A NumPy array containing camera poses and bounding box depths.
- Optional metadata files: `temperature.npy`, `exposure.npy`, `coc.npy`.
- Configuration object `config` with parameters: `factor` (downsampling), `llffhold` (test split frequency), `spherify` (camera path type).

**Outputs:**
- A `utils.Rays` object containing NDC origins, directions, view directions, pixel radii, near/far planes, and auxiliary metadata.
- Split arrays for training rays and render/test rays.

### Data Structures
**`utils.Rays` Object:**
- `origins`: Float32 array of shape `(num_rays, 3)`. NDC coordinates.
- `directions`: Float32 array of shape `(num_rays, 3)`. Normalized direction vectors.
- `viewdirs`: Float32 array of shape `(num_rays, 3)`. Same as directions if `use_viewdirs` is enabled.
- `radii`: Float32 array of shape `(num_rays,)`. Pixel radius for cone sampling.
- `loss_mult`: Float32 array of shape `(num_rays,)`. Usually ones.
- `near`: Float32 array of shape `(num_rays,)`. Near plane distances.
- `far`: Float32 array of shape `(num_rays,)`. Far plane distances.
- `temperature`: Float32 array of shape `(num_rays,)` or `(num_rays, channels)`. Normalized temperature.
- `exposure`: Float32 array of shape `(num_rays,)` or `(num_rays, channels)`. Normalized exposure.
- `coc`: Float32 array of shape `(num_rays,)` or `(num_rays, channels)`. Circle of confusion.

**`poses_bounds.npy` Layout:**
- Raw shape: `(-1, 3, 5)`.
- Reshaped to `[3, 5, N]` where `N` is the number of images.
- The last two columns of the original array (before reshaping) correspond to bounding box depths `bds` (min and max depth along the viewing ray).

### Steps
1. **Load Images:**
   - Determine image directory. If `config.factor > 0`, append `_factor` to the directory name.
   - Load images as float32, normalized to `[0, 1]`.
   - Check for mismatch between number of images and poses. If `poses.shape[-1] != images.shape[-1]`, raise `RuntimeError` [S1, unofficial].

2. **Load Metadata:**
   - Load `poses_bounds.npy`, `temperature.npy`, `exposure.npy`, and `coc.npy`.
   - Reshape temperature, exposure, and coc arrays to match the image channel dimension: `temperature.reshape(images.shape[3], -1)` [S1, unofficial].
   - Normalize metadata:
     - Temperature: `temperature.astype(np.float32) / 10000` [S1, unofficial].
     - Exposure: `(exposure.astype(np.float32) + 3) / 6` [S1, unofficial].

3. **Parse Poses:**
   - Reshape `poses_arr` from `[-1, 3, 5]` to `[3, 5, N]` using `poses_arr[:, :-2].reshape([-1, 3, 5]).transpose([1, 2, 0])` [S1, unofficial].
   - Extract bounding box depths `bds` from the last two columns: `bds = poses_arr[:, -2:].transpose([1, 0])` [S1, unofficial].

4. **Update Intrinsics for Downsampling:**
   - If `config.factor > 1`, update intrinsic parameters.
   - Set `poses[:2, 4, :]` to the downsampled image height and width.
   - Scale `poses[2, 4, :]` (focal length component) by `1. / factor` [S1, unofficial].

5. **Correct Rotation and Axis Convention:**
   - The LLFF dataset uses a specific axis convention that requires correction.
   - Concatenate columns: `[poses[:, 1:2, :], -poses[:, 0:1, :], poses[:, 2:, :]]` [S1, unofficial].
   - Move the variable dimension (images) to axis 0: `poses = np.moveaxis(poses, -1, 0)` [S1, unofficial].

6. **Rescale Scene Geometry:**
   - Calculate scale factor: `scale = 1. / (bds.min() * 0.75)` [S1, unofficial].
   - Apply scale to pose translation vectors: `poses[:, :3, 3] *= scale` [S1, unofficial].
   - Apply scale to bounding depths: `bds *= scale` [S1, unofficial].

7. **Recenter Poses:**
   - Compute the average pose `c2w` using `_poses_avg` (see Formula below).
   - Construct identity bottom row `[0, 0, 0, 1]`.
   - Invert the average pose: `inv_c2w = np.linalg.inv(c2w)` [S1, unofficial].
   - Transform all poses: `poses = inv_c2w @ poses` [S1, unofficial].

8. **Generate Camera Path (Optional):**
   - If `config.spherify` is true or rendering test paths, generate a spiral or spherical path.
   - For spiral/spherical: Compute average pose, up vector, close/inf depths, mean focal length, and radii.
   - Generate `n_views=30` poses using rotation angles `theta` (see Formula below) [S1, unofficial].

9. **Split Dataset:**
   - Test indices: `i_test = np.arange(images.shape[0])[::config.llffhold]` [S1, unofficial].
   - Train indices: All indices not in `i_test`.
   - Select images, poses, and metadata based on the split [S1, unofficial].

10. **Convert to NDC:**
    - For non-spherified data, convert ray origins and directions to NDC using `convert_to_ndc` [S1, unofficial].
    - Calculate pixel radii for mip-NeRF:
      - Compute `dx` and `dy` distances between adjacent ray origins.
      - Average them.
      - Scale by `2 / sqrt(12)` [S1, unofficial].

11. **Construct Rays Object:**
    - Create `utils.Rays` with NDC origins, directions, viewdirs, radii, loss multipliers (ones), near/far planes, and auxiliary metadata [S1, unofficial].

12. **Separate Render Rays:**
    - If in test split, separate render rays from dataset rays at `n_render_poses`.
    - Assign `self.render_rays` and `self.rays`.
    - Update `camtoworlds` to exclude render poses [S1, unofficial].

### Formulas
**Average Pose Construction (`_poses_avg`):**
Let $P$ be the set of camera poses (C2W matrices).
1. Center: $c = \text{mean}(P[:, :3, 3])$
2. View direction: $v_2 = \text{normalize}(\sum P[:, :3, 2])$
3. Up vector: $u = \sum P[:, :3, 1]$
4. Right vector: $v_0 = \text{normalize}(\text{cross}(u, v_2))$
5. True Up vector: $v_1 = \text{normalize}(\text{cross}(v_2, v_0))$
6. View matrix $M = [v_0, v_1, v_2, c]$ (columns) [S1, unofficial].

**Spherical Path Generation:**
1. Centroid: $z_h = \text{centroid}[2]$
2. Radius: $r = \sqrt{\text{mean}(\sum P_{reset}[:, :3, 3]^2)}$
3. Circle radius: $r_{circle} = \sqrt{r^2 - z_h^2}$
4. Camera origin for angle $\theta$: $O(\theta) = [r_{circle} \cos(\theta), r_{circle} \sin(\theta), z_h]$ [S1, unofficial].

### Parameters
- `config.factor`: Downsampling factor. Default 1. If > 0, images are loaded from a subdirectory with suffix `_factor` [S1, unofficial].
- `config.llffhold`: Holdout factor. Determines test set frequency (every N-th image). Default 8 [S1, unofficial].
- `config.spherify`: Boolean flag. If true, generates spherical camera path instead of spiral. Default False [S1, unofficial].

### Edge Cases and Pitfalls
- **Image/Pose Mismatch:** If the number of loaded images does not match the number of poses in `poses_bounds.npy`, a `RuntimeError` is raised [S1, unofficial].
- **Metadata Normalization:** Temperature and exposure values must be normalized as specified. Failure to do so may lead to incorrect lighting modeling [S1, unofficial].
- **Axis Convention:** The LLFF dataset uses a different axis convention than standard OpenGL/COLMAP. The specific column reordering and sign flip in Step 5 are critical for correct pose interpretation [S1, unofficial].
- **NDC Validity:** NDC conversion is only valid for forward-facing scenes. Do not use for non-forward-facing scenes unless `no_ndc` is explicitly handled elsewhere [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of images and pixels for loading and preprocessing. Pose recentering and NDC conversion are $O(N)$ where $N$ is the number of rays.
- **Memory Complexity:** Proportional to the number of rays and auxiliary metadata. The `utils.Rays` object stores all ray data in memory.
- **Strengths:** Handles the specific quirks of the LLFF dataset, including axis correction and metadata normalization. Enables efficient training via NDC and mip-NeRF radii.
- **Limitations:** Only suitable for forward-facing scenes (due to NDC). Requires careful handling of metadata normalization.
- **When to Use:** When training NeRF on the LLFF dataset.
- **When Not to Use:** For non-forward-facing scenes or datasets without the specific LLFF metadata structure.

## Background
The LLFF dataset (Look at Light From) is a widely used benchmark for novel view synthesis, particularly for forward-facing scenes. The preprocessing steps described here are derived from the original NeRF implementation and subsequent adaptations (such as CaSE-NeRF) that handle the specific data format provided by the LLFF authors, which includes COLMAP poses and additional metadata like temperature and exposure [S1, unofficial].

## Sources
[S1] https://github.com/CPREgroup/CaSE-NeRF, https://github.com/CPREgroup/CaSE-NeRF, fetched 2026-10-30 (unofficial)
