# Point Feature Generation


## Overview
Point Feature Generation constructs the initial feature tensors for ray-based volume rendering by indexing spatial data and integrating image-derived attributes [S1, unofficial]. It differs from sibling methods by explicitly separating general input features from color volume data, allowing for flexible integration of auxiliary image features alongside standard geometric ray points [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** `input_feat` (tensor of input features), `img_feat` (optional tensor of image features), `view_ids` (list of view identifiers), `data_dir` (path to image/depth data), `scan` (scene identifier), `light_idx` (lighting index), `downSample` (scaling factor), `remap` (view index mapping), `proj_mats` (precomputed projection matrices), `intrinsics`, `world2cams`, `cam2worlds`.
*   **Outputs:** `ray_feats` (features assigned to rays), `color_volume` (features indexed from images, if used).

**Data Structures:**
*   `ray_feats`: Tensor with 8 channels.
*   `color_volume`: Tensor with remaining channels from `input_feat`.
*   `affine_mat`: List of projection matrices for each view.
*   `affine_mat_inv`: List of inverse projection matrices.
*   `imgs`: List of resized image tensors.
*   `depths_h`: List of height-adjusted depth maps.

**Steps:**
1.  **Initialize Storage:** Create empty lists for `affine_mat`, `affine_mat_inv`, `imgs`, `depths_h`, `proj_mats`, `intrinsics`, `w2cs`, `c2ws`, and `near_fars` [S1, unofficial].
2.  **Iterate Views:** For each `vid` in `view_ids`:
    a.  **Load Image:** Construct filename `rect_{vid + 1:03d}_{light_idx}_r5000.png` in `Rectified/{scan}_train/` [S1, unofficial]. Note: Image IDs are 1-based (1 to 49) [S1, unofficial].
    b.  **Resize:** Load image, resize to `img_wh = np.round(img.size * downSample).astype('int')` using bilinear interpolation, and apply transform [S1, unofficial].
    c.  **Map Indices:** Retrieve `index_mat = remap[vid]` [S1, unofficial].
    d.  **Retrieve Matrices:** Get `proj_mat_ls` and `near_far` from `proj_mats[index_mat]` [S1, unofficial]. Append `intrinsics[index_mat]`, `world2cams[index_mat]`, and `cam2worlds[index_mat]` to their respective lists [S1, unofficial].
    e.  **Affine Matrices:** Append `proj_mat_ls` to `affine_mat` and its inverse `np.linalg.inv(proj_mat_ls)` to `affine_mat_inv` [S1, unofficial].
    f.  **Load Depth:** If `depth_map_{vid:04d}.pfm` exists in `Depths_raw/{scan}/`, read depth, mask, and `depth_h` [S1, unofficial].
3.  **Feature Assignment:**
    a.  **Dimension Check:** If `img_feat` is provided, add its spatial dimensions to `feat_dim` [S1, unofficial].
    b.  **Color Volume Logic:**
        i.  If **not** using color volume: Initialize `input_feat` and index volume features [S1, unofficial].
        ii. If **using** color volume: Directly index volume features [S1, unofficial].
    c.  **Split Features:** Assign the first 8 channels of `input_feat` to `ray_feats` [S1, unofficial]. Assign the remaining channels to the `color_volume` built from images [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_point_features(input_feat, img_feat, view_ids, data_dir, scan, light_idx, downSample, remap, proj_mats, intrinsics, world2cams, cam2worlds):
    affine_mat = []
    affine_mat_inv = []
    imgs = []
    depths_h = []
    proj_mats_ls = []
    near_fars = []
    
    # feat_dim calculation if img_feat is present
    feat_dim = input_feat.shape[1] # Assuming channel-first or similar
    if img_feat is not None:
        feat_dim += img_feat.shape[-1] # Add spatial dimensions as per note [S1, unofficial]

    for vid in view_ids:
        # 1-based ID in filename
        img_filename = os.path.join(data_dir, f'Rectified/{scan}_train/rect_{vid + 1:03d}_{light_idx}_r5000.png')
        img = Image.open(img_filename)
        img_wh = np.round(np.array(img.size) * downSample).astype('int')
        img = img.resize(img_wh, Image.BILINEAR)
        img = transform(img) # [G] Apply standard normalization/transform
        imgs.append(img)

        index_mat = remap[vid]
        proj_mat_ls, near_far = proj_mats[index_mat]
        
        proj_mats_ls.append(proj_mat_ls)
        near_fars.append(near_far)
        intrinsics.append(intrinsics[index_mat])
        world2cams.append(world2cams[index_mat])
        cam2worlds.append(cam2worlds[index_mat])
        
        affine_mat.append(proj_mat_ls)
        affine_mat_inv.append(np.linalg.inv(proj_mat_ls))

        depth_filename = os.path.join(data_dir, f'Depths_raw/{scan}/depth_map_{vid:04d}.pfm')
        if os.path.exists(depth_filename):
            depth, mask, depth_h = read_depth(depth_filename) # [G] Helper to read PFM
            depths_h.append(depth_h)

    # Feature splitting
    # Note: Source implies input_feat is split based on usage of color volume
    if not use_color_volume:
        # Initialize input_feat and index volume features [S1, unofficial]
        ray_feats = input_feat[:, :8]
        color_volume = input_feat[:, 8:]
        # Index volume features logic [S1, unofficial]
    else:
        # Directly index volume features [S1, unofficial]
        ray_feats = input_feat[:, :8]
        color_volume = input_feat[:, 8:]
        
    return ray_feats, color_volume
```

**Parameters:**
*   `downSample`: Scaling factor for image resolution. Typical value: Not given in the sources.
*   `vid`: View identifier. Range: 1 to 49 in filenames [S1, unofficial].

**Edge Cases:**
*   **Missing Depth:** If `depth_map` does not exist, depth data is skipped [S1, unofficial].
*   **1-based Indexing:** Image filenames use 1-based indexing (`vid + 1`), while internal arrays may be 0-based [S1, unofficial].

## Subtopics

### [Voxel Feature Composition](7021-voxel-feature-composition.md)
Generates per-voxel feature vectors containing coordinates, RGB color, and opacity by sampling points within each Gaussian's volume and weighting them by Gaussian attenuation. Choose this when Gaussian splatting is the target primitive.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of views and image resolution due to resizing and loading [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of views and feature dimensions [S1, unofficial].
*   **Strengths:** Flexible integration of image features; separates geometric ray features from color data [S1, unofficial].
*   **Limitations:** Requires precomputed projection matrices and view mappings [S1, unofficial].
*   **When to use:** When building color volumes from images for ray-based rendering [S1, unofficial].
*   **When not to use:** When not using color volumes or when simple voxelization is sufficient [S1, unofficial].

## Background
Originated in the PointNeRF project [S1, unofficial]. It is used in neural radiance field rendering pipelines that utilize point-based features and color volumes [S1, unofficial].

## Sources
[S1] Xharlie/pointnerf, https://github.com/Xharlie/pointnerf, fetched 2026-10-26 (unofficial)
