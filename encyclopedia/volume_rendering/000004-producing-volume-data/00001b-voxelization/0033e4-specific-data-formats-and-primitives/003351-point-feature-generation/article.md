# Point Feature Generation


## Overview
Point Feature Generation creates per-ray feature vectors by indexing volume features and constructing color volumes from input images [S1, unofficial]. It serves as the data preparation step for ray-based volume rendering, transforming raw image and depth inputs into structured feature tensors suitable for neural network processing [S1, unofficial]. It differs from sibling methods by focusing on the aggregation of multi-view image features into ray-aligned tensors rather than generating static voxel grids or point clouds directly.

## How it works
**Inputs:**
- `view_ids`: List of view identifiers (1-based index for file naming).
- `data_dir`: Directory path containing `Rectified` and `Depths_raw` folders.
- `scan`: Scan identifier string.
- `light_idx`: Lighting condition index.
- `downSample`: Scaling factor for image resolution.
- `img_feat`: Optional spatial feature tensor from images.
- `use_color_volume`: Boolean flag indicating whether to build a color volume.

**Outputs:**
- `ray_feats`: Tensor of shape `(N, 8)` containing the first 8 channels of input features for $N$ rays.
- `color_volume`: Optional tensor built from images if `use_color_volume` is true.
- `index_volume`: Volume features indexed for ray access.

**Data Structures:**
- `affine_mat`: List of projection matrices for each view.
- `affine_mat_inv`: List of inverse projection matrices.
- `imgs`: List of resized image tensors.
- `intrinsics`, `w2cs`, `c2ws`: Camera parameters per view.
- `near_fars`: Near and far clipping planes per view.

**Steps:**
1. Initialize empty lists for `affine_mat`, `affine_mat_inv`, `imgs`, `depths_h`, `proj_mats`, `intrinsics`, `w2cs`, `c2ws`, and `near_fars`.
2. For each view ID `vid` in `view_ids`:
   a. Construct `img_filename` using `Rectified/{scan}_train/rect_{vid + 1:03d}_{light_idx}_r5000.png`.
   b. Construct `depth_filename` using `Depths_raw/{scan}/depth_map_{vid:04d}.pfm`.
   c. Load and resize image to `img_wh = round(img.size * downSample)`.
   d. Apply transformation `self.transform(img)` and append to `imgs`.
   e. Retrieve `index_mat = self.remap[vid]`.
   f. Get `proj_mat_ls` and `near_far` from `self.proj_mats[index_mat]`.
   g. Append `self.intrinsics[index_mat]` to `intrinsics`, `self.world2cams[index_mat]` to `w2cs`, and `self.cam2worlds[index_mat]` to `c2ws`.
   h. Append `proj_mat_ls` to `affine_mat` and its inverse to `affine_mat_inv`.
   i. If `depth_filename` exists, read depth, mask, and `depth_h` using `self.read_depth`.
3. Determine `feat_dim`: If `img_feat` is provided, add its spatial dimensions to the base feature dimension.
4. If `use_color_volume` is false:
   a. Initialize `input_feat` and index volume features.
   b. Assign the first 8 channels of `input_feat` to `ray_feats`.
   c. Assign remaining channels to the color volume built from images.
5. If `use_color_volume` is true:
   a. Directly index volume features.

**Parameters:**
- `downSample`: Typical value depends on resolution requirements; not specified in sources [G].
- `light_idx`: Specific to dataset lighting conditions; not specified in sources [G].

**Edge Cases:**
- Depth files may not exist for all views; handled by existence check [S1, unofficial].
- Image file IDs are 1-based, requiring `vid + 1` for filename construction [S1, unofficial].

## Subtopics
## Operating characteristics
Time complexity is linear with respect to the number of views and rays processed. Memory usage depends on the size of the color volume and feature tensors. Performance scales with image resolution and downSample factor. Strengths include efficient multi-view feature aggregation. Limitations include dependency on precise camera calibration and depth data availability. Use when ray-based rendering requires pre-computed image features. Avoid when static voxel representations are sufficient.

## Background
Originated in the context of neural radiance fields and point-based rendering methods, specifically within the PointNeRF framework [S1, unofficial]. It addresses the need to efficiently transfer multi-view image information into a ray-centric feature space for subsequent neural network processing.

## Sources
[S1] Xharlie/pointnerf, https://github.com/Xharlie/pointnerf, fetched 2026-10-26 (unofficial)
